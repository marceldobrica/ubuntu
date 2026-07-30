# Chapter 14 — Traefik

## Overview

Deploy Traefik as the cluster ingress controller. Route HTTP/HTTPS traffic to GitLab (if in-cluster), ArgoCD, and application namespaces. Integrate with cert-manager (chapter 15) for Let's Encrypt certificates.

## Prerequisites

- [Chapter 11 — Kubernetes (K3s)](./11-kubernetes-k3s.md)
- [Chapter 13 — GitOps Repository](./13-gitops-repository.md)
- Ports 80/443 forwarded to MiniPC (chapter 04)

## Goals

- [ ] Traefik running in `traefik` namespace
- [ ] IngressRoute or Ingress resources working
- [ ] HTTP → HTTPS redirect
- [ ] ArgoCD reachable at `argo.<YOUR_DOMAIN>`

## Steps

### 1. Disable K3s default Traefik (if not done)

If K3s was installed with bundled Traefik, reinstall with `--disable traefik` or leave both disabled during migration.

### 2. Install via Helm (GitOps recommended)

Add to `infrastructure/traefik/base/values.yaml`:

```yaml
ports:
  web:
    port: 8000
    expose:
      default: true
    exposedPort: 80
  websecure:
    port: 8443
    expose:
      default: true
    exposedPort: 443

providers:
  kubernetesCRD:
    enabled: true
  kubernetesIngress:
    enabled: true

service:
  type: LoadBalancer
  # On bare metal, K3s servicelb maps 80/443 to node
```

Install manually for first test:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  -f infrastructure/traefik/base/values.yaml
```

### 3. Pin Traefik to host ports (single node)

K3s ServiceLB binds LoadBalancer services to node ports 80/443 automatically. Verify:

```bash
kubectl get svc -n traefik
```

### 4. IngressRoute for ArgoCD

Example `infrastructure/argocd/ingress.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`argo.<YOUR_DOMAIN>`)
      kind: Rule
      services:
        - name: argocd-server
          port: 443
          scheme: https
          serversTransport: argocd-insecure
  tls:
    secretName: wildcard-<YOUR_DOMAIN>-tls
```

ArgoCD server runs with TLS internally — use `serversTransport` or configure ArgoCD `--insecure` behind Traefik.

Apply via GitOps or:

```bash
kubectl apply -f infrastructure/argocd/ingress.yaml
```

### 5. HTTP to HTTPS redirect

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: redirect-http
  namespace: traefik
spec:
  entryPoints:
    - web
  routes:
    - match: HostRegexp(`{host:.+}`)
      kind: Rule
      middlewares:
        - name: redirect-https
      services:
        - name: noop@internal
          kind: TraefikService
```

With middleware `redirectScheme` to https.

### 6. GitLab ingress note

GitLab runs **outside** K3s (Omnibus on host) in this guide. Traefik routes **in-cluster** apps. GitLab uses its own nginx/LE or sits behind Traefik via external Service — pick one:

- **A:** GitLab Omnibus LE on `gitlab.<YOUR_DOMAIN>` (chapter 08)
- **B:** Traefik TCP/HTTPS passthrough to host IP (advanced)

Document your choice in the GitOps README.

## Verify

```bash
kubectl get pods -n traefik
kubectl get ingressroute -A
curl -Ik https://argo.<YOUR_DOMAIN>
```

Browser: ArgoCD UI loads (cert may warn until chapter 15).

## Troubleshooting

| Problem | Fix |
|---------|-----|
| 404 on all routes | Check IngressRoute host matches DNS |
| Port 80 in use | `sudo ss -tlnp | grep :80` — GitLab nginx conflict |
| ArgoCD redirect loop | Set `server.insecure: true` in argocd-cmd-params |

## Next

→ [Chapter 15 — Cert Manager](./15-cert-manager.md)
