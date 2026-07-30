# Chapter 11 — Kubernetes (K3s)

## Overview

Install K3s as a single-node cluster on the MiniPC. K3s includes Traefik by default — we will customize or replace it in chapter 14.

## Prerequisites

- [Chapter 06 — Docker](./06-docker.md) (K3s uses containerd; Docker coexists)
- Sufficient RAM (8 GB+ after GitLab)

## Goals

- [ ] K3s server running
- [ ] `kubectl` configured for `labadmin`
- [ ] Base namespaces created
- [ ] Cluster healthy

## Steps

### 1. Install K3s server

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --tls-san gitlab.<YOUR_DOMAIN> \
  --tls-san argo.<YOUR_DOMAIN>
```

Add `--disable traefik` if you plan a fully custom Traefik install in chapter 14:

```bash
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable traefik
```

### 2. Configure kubectl for labadmin

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown labadmin:labadmin ~/.kube/config
chmod 600 ~/.kube/config
```

Optional — set context:

```bash
kubectl config rename-context default lab-minipc
```

### 3. Verify cluster

```bash
kubectl get nodes
kubectl get pods -A
```

### 4. Create namespaces

```bash
kubectl create namespace argocd
kubectl create namespace traefik
kubectl create namespace cert-manager
kubectl create namespace drupal
kubectl create namespace wordpress
kubectl create namespace symfony
kubectl create namespace monitoring
```

Label for GitOps (optional):

```bash
kubectl label namespace drupal app.kubernetes.io/part-of=lab-platform
```

### 5. Install Helm (used by ArgoCD, cert-manager)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 6. Resource note (single node)

GitLab + K3s on one machine is tight. Monitor:

```bash
kubectl top nodes  # after metrics-server installed
free -h
```

Consider disabling GitLab bundled Prometheus (chapter 08) to save RAM.

## Verify

```bash
kubectl cluster-info
kubectl get ns
kubectl get pods -A | grep -v Running | grep -v Completed
# No unexpected CrashLoopBackOff
```

```bash
sudo systemctl status k3s
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| kubectl connection refused | Check `k3s` service; verify kubeconfig |
| Port 6443 conflict | `sudo lsof -i :6443` |
| Pods pending (CPU/mem) | Reduce GitLab footprint or add worker (ch 23) |

## Next

→ [Chapter 12 — ArgoCD](./12-argocd.md)
