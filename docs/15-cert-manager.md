# Chapter 15 — Cert Manager

## Overview

Install cert-manager and configure Let's Encrypt certificates using Cloudflare DNS-01 challenge. Issue a wildcard cert for `*.<YOUR_DOMAIN>` usable by Traefik ingresses.

## Prerequisites

- [Chapter 05 — Cloudflare](./05-cloudflare.md) — API token ready
- [Chapter 14 — Traefik](./14-traefik.md)

## Goals

- [ ] cert-manager installed
- [ ] ClusterIssuer for Let's Encrypt staging (test) then production
- [ ] Wildcard certificate issued
- [ ] Traefik using TLS secret

## Steps

### 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=300s
```

Or via Helm/GitOps (preferred long-term).

### 2. Create Cloudflare API token secret

```bash
kubectl create secret generic cloudflare-api-token \
  -n cert-manager \
  --from-literal=api-token=<CLOUDFLARE_API_TOKEN>
```

### 3. ClusterIssuer — staging (test first)

`infrastructure/cert-manager/base/clusterissuer-staging.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: you@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

```bash
kubectl apply -f infrastructure/cert-manager/base/clusterissuer-staging.yaml
```

### 4. Wildcard Certificate (staging test)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-<YOUR_DOMAIN>
  namespace: traefik
spec:
  secretName: wildcard-<YOUR_DOMAIN>-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - "<YOUR_DOMAIN>"
    - "*.<YOUR_DOMAIN>"
```

```bash
kubectl apply -f infrastructure/cert-manager/overlays/minipc/certificate-staging.yaml
kubectl describe certificate -n traefik wildcard-<YOUR_DOMAIN>
kubectl get challenges -A
```

Fix any DNS/API errors before production.

### 5. ClusterIssuer — production

Duplicate issuer with:

```yaml
    server: https://acme-v02.api.letsencrypt.org/directory
  name: letsencrypt-prod
```

Update Certificate `issuerRef` to `letsencrypt-prod`.

### 6. Reference cert in Traefik IngressRoutes

```yaml
  tls:
    secretName: wildcard-<YOUR_DOMAIN>-tls
```

Secret must be in same namespace as IngressRoute, or use cross-namespace reference / copy secret.

### 7. Cloudflare SSL mode

After valid origin certs: set Cloudflare to **Full (strict)** if using orange cloud proxy.

## Verify

```bash
kubectl get clusterissuer
kubectl get certificate -A
kubectl describe certificate -n traefik wildcard-<YOUR_DOMAIN>
```

```bash
curl -vI https://argo.<YOUR_DOMAIN> 2>&1 | grep -i issuer
# Let's Encrypt after prod issuer
echo | openssl s_client -connect argo.<YOUR_DOMAIN>:443 -servername argo.<YOUR_DOMAIN> 2>/dev/null | openssl x509 -noout -dates
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Challenge pending | Token permissions; zone ID; DNS propagation |
| Rate limit | Use staging issuer for tests |
| Wrong cert served | Secret namespace; IngressRoute tls.secretName |

## Next

→ [Chapter 16 — Drupal Platform](./16-drupal-platform.md)
