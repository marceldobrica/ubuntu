# Chapter 16 — Drupal Platform

## Overview

Deploy a reference Drupal site in the `drupal` namespace: MariaDB, Redis, persistent storage, PHP/Apache or FPM/nginx container, Service, and Traefik Ingress at `drupal.<YOUR_DOMAIN>`.

## Prerequisites

- [Chapter 13 — GitOps Repository](./13-gitops-repository.md)
- [Chapter 14 — Traefik](./14-traefik.md)
- [Chapter 15 — Cert Manager](./15-cert-manager.md)

## Goals

- [ ] MariaDB with PVC
- [ ] Redis for caching
- [ ] Drupal deployment running
- [ ] HTTPS ingress working
- [ ] Site installable via browser

## Steps

### 1. Namespace and secrets

```bash
kubectl create namespace drupal --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic drupal-db \
  -n drupal \
  --from-literal=mysql-root-password=<ROOT_PASSWORD> \
  --from-literal=mysql-password=<DRUPAL_DB_PASSWORD> \
  --from-literal=mysql-user=drupal \
  --from-literal=mysql-database=drupal

kubectl create secret docker-registry gitlab-registry \
  -n drupal \
  --docker-server=registry.<YOUR_DOMAIN> \
  --docker-username=<REGISTRY_USER> \
  --docker-password=<REGISTRY_TOKEN>
```

### 2. MariaDB (GitOps manifest sketch)

`apps-platform/drupal/base/mariadb.yaml` — Deployment + Service + PVC (10Gi+):

```yaml
# Use official mariadb:11 image or bitnami/mariadb helm chart
# StorageClass: local-path (K3s default)
```

K3s includes `local-path` provisioner by default.

### 3. Redis

```yaml
# redis:7-alpine — Deployment + Service, no PVC required for cache
```

### 4. Drupal deployment

Options:

- **A:** Official `drupal` Docker image + custom settings via ConfigMap
- **B:** Custom image built in CI (chapter 19) with composer dependencies

Example env:

```yaml
env:
  - name: DRUPAL_DB_HOST
    value: mariadb
  - name: DRUPAL_DB_NAME
    valueFrom:
      secretKeyRef:
        name: drupal-db
        key: mysql-database
  - name: DRUPAL_DB_USER
    valueFrom:
      secretKeyRef:
        name: drupal-db
        key: mysql-user
  - name: DRUPAL_DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: drupal-db
        key: mysql-password
```

Mount PVC for `sites/default/files`.

### 5. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: drupal
  namespace: drupal
spec:
  selector:
    app: drupal
  ports:
    - port: 80
      targetPort: 80
```

### 6. IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: drupal
  namespace: drupal
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`drupal.<YOUR_DOMAIN>`)
      kind: Rule
      services:
        - name: drupal
          port: 80
  tls:
    secretName: wildcard-<YOUR_DOMAIN>-tls
```

Copy TLS secret to `drupal` namespace or use cert-manager Certificate in that namespace.

### 7. ArgoCD sync

Commit manifests to `apps-platform/drupal/` and sync:

```bash
argocd app sync drupal
kubectl get pods -n drupal
```

### 8. DNS

Cloudflare A record: `drupal.<YOUR_DOMAIN>` → public IP (chapter 05).

## Verify

```bash
kubectl get all -n drupal
kubectl logs -n drupal -l app=drupal --tail=50
curl -Ik https://drupal.<YOUR_DOMAIN>
```

Browser: Drupal installer or existing site loads over HTTPS.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| DB connection refused | Service name; wait for MariaDB ready |
| 502 from Traefik | Pod not ready; check probes |
| File upload errors | PVC mount permissions on files directory |

## Next

→ [Chapter 17 — WordPress Platform](./17-wordpress-platform.md)
