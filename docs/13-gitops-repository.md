# Chapter 13 — GitOps Repository

## Overview

Create a GitOps repository in GitLab that holds Kubernetes manifests for Traefik, cert-manager, ArgoCD apps, and each CMS platform. ArgoCD watches this repo and syncs the cluster.

## Prerequisites

- [Chapter 12 — ArgoCD](./12-argocd.md)
- [Chapter 08 — GitLab CE](./08-gitlab-ce.md)

## Goals

- [ ] `lab/gitops` repository created
- [ ] Directory structure defined
- [ ] ArgoCD Application manifests for each component
- [ ] Kustomize bases/overlays pattern

## Steps

### 1. Create repository

In GitLab: **lab/gitops** — initialize with README.

Clone locally:

```bash
git clone git@gitlab.<YOUR_DOMAIN>:lab/gitops.git
cd gitops
```

### 2. Repository structure

```
gitops/
├── README.md
├── apps/                          # ArgoCD Application CRs
│   ├── traefik.yaml
│   ├── cert-manager.yaml
│   ├── drupal.yaml
│   ├── wordpress.yaml
│   └── symfony.yaml
├── clusters/
│   └── minipc/                    # Single-cluster overlay
│       └── kustomization.yaml
├── infrastructure/
│   ├── traefik/
│   │   ├── base/
│   │   └── overlays/minipc/
│   ├── cert-manager/
│   │   ├── base/
│   │   └── overlays/minipc/
│   └── argocd/
│       └── ingress.yaml
└── apps-platform/
    ├── drupal/
    │   ├── base/
    │   └── overlays/minipc/
    ├── wordpress/
    │   ├── base/
    │   └── overlays/minipc/
    └── symfony/
        ├── base/
        └── overlays/minipc/
```

### 3. Root kustomization (example)

`clusters/minipc/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/traefik.yaml
  - ../../apps/cert-manager.yaml
  - ../../apps/drupal.yaml
```

### 4. ArgoCD Application example

`apps/drupal.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: drupal
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@gitlab.<YOUR_DOMAIN>:lab/gitops.git
    targetRevision: main
    path: apps-platform/drupal/overlays/minipc
  destination:
    server: https://kubernetes.default.svc
    namespace: drupal
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 5. Bootstrap ArgoCD app-of-apps (optional)

Single root Application pointing at `clusters/minipc`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lab-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@gitlab.<YOUR_DOMAIN>:lab/gitops.git
    targetRevision: main
    path: clusters/minipc
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply once:

```bash
kubectl apply -f apps/root-app.yaml
```

### 6. Secrets strategy

Do **not** commit plaintext secrets. Use:

- Sealed Secrets
- External Secrets Operator + GitLab variables
- SOPS-encrypted files

Document chosen approach in chapter 22. Placeholder secrets via `kubectl create secret` for lab.

### 7. Commit and push

```bash
git add .
git commit -m "Initial GitOps structure"
git push origin main
```

## Verify

```bash
argocd app list
argocd app get drupal
kubectl get applications -n argocd
```

ArgoCD UI shows applications (may be OutOfSync until manifests complete in later chapters).

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Application invalid path | Verify repo path matches GitLab structure |
| SSH repo access | Add deploy key or ArgoCD repo credential |
| Sync loop | Check manifest errors in ArgoCD UI |

## Next

→ [Chapter 14 — Traefik](./14-traefik.md)
