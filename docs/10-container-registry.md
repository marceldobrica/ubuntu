# Chapter 10 — Container Registry

## Overview

Use GitLab's integrated Container Registry to store images built by CI. Push a first test image and verify pull from the MiniPC and Kubernetes (later chapters).

## Prerequisites

- [Chapter 08 — GitLab CE](./08-gitlab-ce.md)
- [Chapter 09 — GitLab Runner](./09-gitlab-runner.md)

## Goals

- [ ] Registry reachable at `registry.<YOUR_DOMAIN>` (or GitLab embedded path)
- [ ] Docker login succeeds
- [ ] First image pushed and visible in GitLab UI

## Steps

### 1. Confirm registry URL

GitLab registry URL formats:

```
registry.<YOUR_DOMAIN>/<group>/<project>:<tag>
# or
gitlab.<YOUR_DOMAIN>:5050/<group>/<project>:<tag>
```

Check in GitLab: **Deploy → Container Registry** for exact path.

### 2. Create deploy token or use personal access token

**Settings → Access Tokens** (project or group):

- Scopes: `read_registry`, `write_registry`

Or **Settings → Repository → Deploy tokens**.

Save `<REGISTRY_USER>` and `<REGISTRY_TOKEN>`.

### 3. Docker login

```bash
docker login registry.<YOUR_DOMAIN> -u <REGISTRY_USER> -p <REGISTRY_TOKEN>
```

Credentials stored in `~/.docker/config.json`.

### 4. Push first image

```bash
docker pull alpine:3.19
docker tag alpine:3.19 registry.<YOUR_DOMAIN>/lab/gitops/alpine:test
docker push registry.<YOUR_DOMAIN>/lab/gitops/alpine:test
```

Adjust path to match your group/project.

### 5. CI push example (snippet for chapter 19)

```yaml
build:
  stage: build
  tags: [minipc]
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE: registry.<YOUR_DOMAIN>/lab/drupal-site/app
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE:$CI_COMMIT_SHA .
    - docker push $IMAGE:$CI_COMMIT_SHA
```

### 6. Kubernetes pull secret (preview)

K3s will need an `imagePullSecret` — created in chapter 13/16:

```bash
kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.<YOUR_DOMAIN> \
  --docker-username=<REGISTRY_USER> \
  --docker-password=<REGISTRY_TOKEN> \
  -n drupal
```

## Verify

```bash
docker pull registry.<YOUR_DOMAIN>/lab/gitops/alpine:test
```

GitLab UI → project → Container Registry shows the tag.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| x509 certificate error | Align TLS with GitLab or Traefik; check Cloudflare mode |
| denied: access forbidden | Token scopes; project path must match |
| 413 entity too large | Increase `registry['storage']` limits in gitlab.rb |

## Next

→ [Chapter 11 — Kubernetes (K3s)](./11-kubernetes-k3s.md)
