# Chapter 22 — Security

## Overview

Harden secrets management, Kubernetes RBAC, and network policies for the lab. Dev environment accessible from internet requires deliberate controls.

## Prerequisites

- [Chapter 02 — Initial Server Hardening](./02-initial-server-hardening.md)
- Cluster and apps running (chapters 11–19)

## Goals

- [ ] No plaintext secrets in Git
- [ ] GitLab 2FA enforced for users
- [ ] RBAC least privilege for CI/deploy accounts
- [ ] Network policies limiting pod traffic
- [ ] Regular secret rotation plan

## Steps

### 1. Secrets in Git — do not

Use one of:

| Tool | Use case |
|------|----------|
| Sealed Secrets | Encrypt secrets committed to GitOps |
| External Secrets | Pull from GitLab CI variables / Vault |
| SOPS + age | Encrypted YAML in repo |

Example Sealed Secrets:

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

kubectl create secret generic drupal-db --dry-run=client -o yaml \
  --from-literal=mysql-password=xxx -n drupal | \
  kubeseal -o yaml > apps-platform/drupal/base/sealed-db.yaml
```

### 2. GitLab CI variables

Store `GITOPS_SSH_KEY`, registry tokens as **masked**, **protected** variables.

Never log secrets in pipeline output.

### 3. Kubernetes RBAC

Create dedicated ServiceAccount for CI deploy (if using kubectl instead of Git push):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deploy
  namespace: symfony
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deploy
  namespace: symfony
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "patch"]
```

Prefer GitOps (no cluster credentials in CI).

### 4. ArgoCD RBAC

Edit `argocd-rbac-cm` — restrict who can sync/delete production apps.

### 5. Network policies

Default deny ingress within namespace, allow from Traefik namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: drupal-from-traefik
  namespace: drupal
spec:
  podSelector:
    matchLabels:
      app: drupal
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: traefik
      ports:
        - port: 80
```

Label Traefik namespace accordingly.

Deny cross-namespace DB access except from app pods.

### 6. Pod security

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true  # where compatible
```

GitLab Runner `privileged` is a known exception — isolate runner on dedicated node when scaling.

### 7. Cloudflare access (optional)

Cloudflare Zero Trust — require authentication for `argo.<YOUR_DOMAIN>` and GitLab admin paths.

### 8. GitLab account protection

Enable 2FA for the initial `root` account and every administrator immediately
after GitLab installation (chapter 08). Create a separate named administrator
for routine use, store recovery codes securely, and enforce 2FA for all users
from **Admin Area → Settings → General → Sign-in restrictions**.

Use SSH keys for Git operations from outside the intranet. For HTTPS Git access,
use scoped, expiring personal access tokens rather than account passwords.

### 9. Audit and updates

```bash
kubectl auth can-i --list --as=system:serviceaccount:symfony:ci-deploy -n symfony
sudo apt update && sudo apt list --upgradable
```

Rotate Cloudflare API token and GitLab tokens periodically.

## Verify

```bash
# No secrets in git history (sample scan)
git grep -i password apps-platform/ || echo "No plaintext passwords found"

kubectl get networkpolicies -A
kubectl get sealedsecrets -A  # if using Sealed Secrets
```

Pen-test checklist:

- [ ] SSH password login disabled (chapter 02)
- [ ] GitLab 2FA enabled and enforced (chapter 08)
- [ ] ArgoCD not open without auth
- [ ] DB ports not exposed via NodePort/LoadBalancer

## Troubleshooting

| Problem | Fix |
|---------|-----|
| App broken after NetworkPolicy | Allow required egress to DNS, DB |
| Sealed secret won't decrypt | Cluster-wide cert mismatch; re-seal |

## Next

→ [Chapter 23 — Scaling](./23-scaling.md)
