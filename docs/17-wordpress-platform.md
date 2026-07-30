# Chapter 17 — WordPress Platform

## Overview

Deploy WordPress in the `wordpress` namespace with MariaDB, persistent uploads volume, and HTTPS at `wp.<YOUR_DOMAIN>`.

## Prerequisites

- [Chapter 16 — Drupal Platform](./16-drupal-platform.md) (pattern reference)
- Chapters 13–15 complete

## Goals

- [ ] WordPress + MariaDB running
- [ ] PVC for wp-content/uploads
- [ ] Ingress with TLS
- [ ] Admin install via browser

## Steps

### 1. Namespace and secrets

```bash
kubectl create namespace wordpress --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic wordpress-db \
  -n wordpress \
  --from-literal=WORDPRESS_DB_HOST=mariadb \
  --from-literal=WORDPRESS_DB_NAME=wordpress \
  --from-literal=WORDPRESS_DB_USER=wordpress \
  --from-literal=WORDPRESS_DB_PASSWORD=<WP_DB_PASSWORD>

kubectl create secret generic mariadb-root \
  -n wordpress \
  --from-literal=MYSQL_ROOT_PASSWORD=<ROOT_PASSWORD> \
  --from-literal=MYSQL_DATABASE=wordpress \
  --from-literal=MYSQL_USER=wordpress \
  --from-literal=MYSQL_PASSWORD=<WP_DB_PASSWORD>
```

### 2. MariaDB

Reuse pattern from Drupal — separate instance per namespace for isolation (lab-appropriate).

`apps-platform/wordpress/base/mariadb.yaml`

### 3. WordPress deployment

Official image `wordpress:6-apache` or custom image from CI.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:6-apache
          envFrom:
            - secretRef:
                name: wordpress-db
          volumeMounts:
            - name: wp-content
              mountPath: /var/www/html/wp-content
      volumes:
        - name: wp-content
          persistentVolumeClaim:
            claimName: wordpress-pvc
```

### 4. PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
```

### 5. Service and IngressRoute

Service on port 80; IngressRoute host `wp.<YOUR_DOMAIN>` with wildcard TLS secret.

### 6. GitOps and ArgoCD

Add `apps/wordpress.yaml` Application; path `apps-platform/wordpress/overlays/minipc`.

### 7. Optional — Redis object cache

Install Redis + `redis-cache` plugin via custom image or init container for performance.

## Verify

```bash
kubectl get pods -n wordpress
curl -Ik https://wp.<YOUR_DOMAIN>
```

Complete WordPress 5-minute install in browser.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Error establishing database connection | Verify secret keys match MariaDB env |
| Permalink 404 | Apache mod_rewrite; `.htaccess` in image |
| Slow performance | Increase PHP resources; add Redis |

## Next

→ [Chapter 18 — Symfony Platform](./18-symfony-platform.md)
