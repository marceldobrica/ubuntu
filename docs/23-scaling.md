# Chapter 23 — Scaling

## Overview

Expand the lab from one MiniPC to multiple nodes: add K3s workers, optionally HA control plane, and distribute workload while keeping GitLab on the first node.

## Prerequisites

- [Chapter 11 — Kubernetes (K3s)](./11-kubernetes-k3s.md)
- Stable single-node cluster
- Additional MiniPCs with Ubuntu 24 installed (chapters 01–03)

## Goals

- [ ] Second node joined as K3s agent
- [ ] Workloads schedulable on workers
- [ ] Optional third worker
- [ ] HA control plane plan documented

## Steps

### 1. Prepare worker nodes

On each new MiniPC:

- Complete chapters 01–03 (OS, hardening, static IP)
- Unique hostname: `lab-node-02`, `lab-node-03`
- Same network reachability to server node

### 2. Get K3s join token (on server node)

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Server IP: `<SERVER_LAN_IP>` (e.g. 192.168.1.100)

### 3. Join second worker

On `lab-node-02`:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_LAN_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```

Verify from server:

```bash
kubectl get nodes -o wide
```

### 4. Label workers

```bash
kubectl label node lab-node-02 node-role.kubernetes.io/worker=worker
```

### 5. Schedule workloads on workers

GitLab stays on node 01 (not in K3s). For heavy apps:

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: node-role.kubernetes.io/worker
              operator: Exists
```

Or taint control plane node (optional):

```bash
kubectl taint nodes lab-node-01 node-role.kubernetes.io/control-plane=true:NoSchedule
```

Remove default K3s taint if present and you want apps on server:

```bash
kubectl taint nodes lab-node-01 node-role.kubernetes.io/control-plane:NoSchedule-
```

### 6. Add third worker

Repeat join steps on `lab-node-03`.

### 7. HA control plane (advanced)

K3s supports embedded etcd HA with 3 server nodes:

| Setup | Nodes | Notes |
|-------|-------|-------|
| Single server | 1 server + N agents | Current lab |
| Embedded etcd HA | 3 servers | Odd number; quorum |
| External DB | 2+ servers + PostgreSQL/MySQL | GitLab already uses PostgreSQL |

Example join additional **server** (not agent):

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=<TOKEN> sh -s - server \
  --server https://<FIRST_SERVER_IP>:6443
```

Requires planning: GitLab co-location, IP stability, load balancer for API.

For cheap hosting mimic: start with 1 server + 2 agents; move to 3-node HA when uptime matters.

### 8. Distributed storage note

`local-path` PVCs bind to one node — pods must run on that node or use shared storage (Longhorn, NFS) for multi-node apps.

For lab: keep DB and app on same node or migrate to Longhorn:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml
```

## Verify

```bash
kubectl get nodes
kubectl describe node lab-node-02 | grep -A5 Allocatable
```

Deploy test pod with anti-affinity; confirm scheduling on worker:

```bash
kubectl run test-nginx --image=nginx --restart=Never
kubectl get pod test-nginx -o wide
kubectl delete pod test-nginx
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Worker NotReady | Firewall between nodes; check 6443, flannel ports |
| Pod pending | PVC node binding; insufficient resources |
| Split brain (HA) | Ensure odd server count; stable IPs |

## Next

→ [Chapter 24 — Troubleshooting](./24-troubleshooting.md)
