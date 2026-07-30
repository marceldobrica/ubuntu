# Chapter 12 — ArgoCD

## Overview

Install ArgoCD in the `argocd` namespace for GitOps continuous delivery. Access the UI at `argo.<YOUR_DOMAIN>` after ingress is configured (chapter 14).

## Prerequisites

- [Chapter 11 — Kubernetes (K3s)](./11-kubernetes-k3s.md)
- Helm installed

## Goals

- [ ] ArgoCD installed in cluster
- [ ] Admin login credentials retrieved
- [ ] CLI optional but recommended
- [ ] Ready to connect GitOps repo (chapter 13)

## Steps

### 1. Install ArgoCD

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for pods:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

### 2. Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Username: `admin`

Change password after first login (UI or CLI).

### 3. Install ArgoCD CLI (optional)

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
argocd version --client
```

### 4. Port-forward for initial access (before Traefik)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Browse `https://localhost:8080` (accept self-signed cert).

### 5. Register GitLab repository (after chapter 13)

```bash
argocd login localhost:8080 --username admin --password '<PASSWORD>' --insecure

argocd repo add git@gitlab.<YOUR_DOMAIN>:lab/gitops.git \
  --ssh-private-key-path ~/.ssh/id_ed25519_gitlab \
  --insecure-ignore-host-key
```

Or HTTPS with token.

### 6. Ingress placeholder

Full HTTPS via Traefik in chapter 14. Hostname target: `argo.<YOUR_DOMAIN>`.

## Verify

```bash
kubectl get svc -n argocd
kubectl get applications -n argocd
argocd account get-user-info  # after login
```

UI: login as admin, dashboard loads.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Pods CrashLoop | Check `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server` |
| Repo connection failed | SSH key in ArgoCD; GitLab host key |
| Out of sync always | Normal until GitOps repo connected |

## Next

→ [Chapter 13 — GitOps Repository](./13-gitops-repository.md)
