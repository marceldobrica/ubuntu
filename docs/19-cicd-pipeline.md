# Chapter 19 — CI/CD Pipeline

## Overview

Wire GitLab CI to build, test, push images, and trigger ArgoCD deployments for Drupal, WordPress, and Symfony projects — the full path from commit to running pods.

## Prerequisites

- [Chapter 09 — GitLab Runner](./09-gitlab-runner.md)
- [Chapter 10 — Container Registry](./10-container-registry.md)
- [Chapter 12 — ArgoCD](./12-argocd.md)
- [Chapter 13 — GitOps Repository](./13-gitops-repository.md)
- At least one app platform chapter (16–18)

## Goals

- [ ] `.gitlab-ci.yml` with build, test, push stages
- [ ] Image tag updated in GitOps repo
- [ ] ArgoCD syncs new version automatically
- [ ] End-to-end deploy verified

## Steps

### 1. Pipeline stages (standard pattern)

```yaml
stages:
  - test
  - build
  - deploy
```

### 2. Symfony example — full pipeline

`.gitlab-ci.yml` in **lab/symfony-app**:

```yaml
variables:
  IMAGE: registry.<YOUR_DOMAIN>/lab/symfony-app
  DOCKER_TLS_CERTDIR: "/certs"

stages:
  - test
  - build
  - deploy

test:
  stage: test
  tags: [minipc]
  image: registry.<YOUR_DOMAIN>/lab/symfony-app/ci:latest
  script:
    - composer install
    - php bin/phpunit

build:
  stage: build
  tags: [minipc]
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE:$CI_COMMIT_SHA -t $IMAGE:latest .
    - docker push $IMAGE:$CI_COMMIT_SHA
    - docker push $IMAGE:latest
  only:
    - main

deploy:
  stage: deploy
  tags: [minipc]
  image: alpine/git
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$GITOPS_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && ssh-keyscan gitlab.<YOUR_DOMAIN> >> ~/.ssh/known_hosts
  script:
    - git clone git@gitlab.<YOUR_DOMAIN>:lab/gitops.git
    - cd gitops
    - 'sed -i "s|image: .*symfony-app:.*|image: '"$IMAGE:$CI_COMMIT_SHA"'|" apps-platform/symfony/base/deployment.yaml'
    - git config user.email "ci@gitlab.<YOUR_DOMAIN>"
    - git config user.name "GitLab CI"
    - git commit -am "Deploy symfony $CI_COMMIT_SHA"
    - git push origin main
  only:
    - main
```

Add `GITOPS_SSH_KEY` as masked CI variable (deploy key with write access to gitops).

### 3. ArgoCD automated sync

Already configured in chapter 13 — sync happens within ~3 minutes or trigger:

```bash
argocd app sync symfony
```

### 4. Alternative — Argo CD Image Updater

Annotate Application to track `:latest` or semver tags without Git commits:

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=registry.<YOUR_DOMAIN>/lab/symfony-app
    argocd-image-updater.argoproj.io/app.update-strategy: latest
```

Install argocd-image-updater separately if desired.

### 5. Drupal / WordPress pipelines

Similar build stage; custom Dockerfiles with composer/wp-cli:

- **Drupal:** `composer install`, config sync, custom image
- **WordPress:** theme/plugin build steps, `composer` if using Bedrock

Deploy stage patches GitOps manifest image tag per app.

### 6. Protected branches and environments

GitLab **Settings → Repository → Protected branches** — protect `main`.

Optional **environments** `staging` / `production` with manual deploy jobs later.

## Verify

1. Push a commit to `symfony-app` main
2. Pipeline passes all stages
3. GitOps repo shows new commit with image tag
4. ArgoCD application Healthy and Synced
5. Running pod uses new image:

```bash
kubectl get pods -n symfony -o jsonpath='{.items[0].spec.containers[0].image}'
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Deploy job cannot push gitops | Deploy key write access |
| ArgoCD not syncing | Automated sync enabled; check diff |
| Old image still running | Rollout status; imagePullPolicy Always |

## Next

→ [Chapter 20 — Monitoring](./20-monitoring.md)
