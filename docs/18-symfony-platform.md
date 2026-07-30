# Chapter 18 — Symfony Platform

## Overview

Deploy a Symfony PHP application in the `symfony` namespace. Symfony fits the CI/CD story well: build a production Docker image in GitLab CI, push to registry, deploy via ArgoCD.

## Prerequisites

- [Chapter 17 — WordPress Platform](./17-wordpress-platform.md)
- Chapters 10, 13, 14, 15 complete

## Goals

- [ ] Symfony app image built and pushed (or use sample)
- [ ] PostgreSQL or MariaDB backend
- [ ] PHP-FPM + nginx sidecar or single nginx+php image
- [ ] HTTPS at `symfony.<YOUR_DOMAIN>`

## Steps

### 1. Sample application repo

In GitLab: **lab/symfony-app** — Symfony skeleton or existing project.

Minimal `Dockerfile`:

```dockerfile
FROM php:8.3-fpm-alpine AS base
RUN apk add --no-cache icu-dev postgresql-dev \
  && docker-php-ext-install intl pdo_pdo_pgsql opcache
WORKDIR /app
COPY . .
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
  && composer install --no-dev --optimize-autoloader

FROM nginx:alpine
COPY --from=base /app /app
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
# php-fpm via separate container or supervisord
```

Adapt to your Symfony version and runtime preference.

### 2. Namespace and database

```bash
kubectl create namespace symfony --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic symfony-db \
  -n symfony \
  --from-literal=DATABASE_URL="postgresql://symfony:<PASSWORD>@postgres:5432/symfony?serverVersion=16&charset=utf8"
```

PostgreSQL Deployment + PVC in `apps-platform/symfony/base/postgres.yaml`.

### 3. Deployment

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: gitlab-registry
      containers:
        - name: php
          image: registry.<YOUR_DOMAIN>/lab/symfony-app:latest
          envFrom:
            - secretRef:
                name: symfony-db
        - name: nginx
          image: nginx:alpine
          # mount shared volume or config for symfony public/
```

Or single container with FrankenPHP/Caddy for simplicity in lab.

### 4. Migrations job

Kubernetes Job or init container:

```yaml
command: ["php", "bin/console", "doctrine:migrations:migrate", "--no-interaction"]
```

Run on each deploy via ArgoCD PreSync hook (optional).

### 5. IngressRoute

Host: `symfony.<YOUR_DOMAIN>`

### 6. Environment-specific config

Use ConfigMap for non-secret env; secrets for `APP_SECRET`, database URL.

Symfony prod checklist:

```bash
php bin/console cache:clear --env=prod
php bin/console assets:install public --env=prod
```

Bake into Docker image in CI.

## Verify

```bash
kubectl get pods -n symfony
curl -Ik https://symfony.<YOUR_DOMAIN>
```

Browser or:

```bash
curl -s https://symfony.<YOUR_DOMAIN> | head
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| 500 APP_ENV | Set `APP_ENV=prod` `APP_DEBUG=0` |
| Migration fails | DB reachable; Job logs |
| Asset 404 | nginx root must point to `public/` |

## Next

→ [Chapter 19 — CI/CD Pipeline](./19-cicd-pipeline.md)
