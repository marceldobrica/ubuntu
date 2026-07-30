# Chapter 25 — Appendix

## Overview

Quick reference cheat sheets for Linux, kubectl, Docker, and Git used throughout the lab.

---

## Useful Linux commands

```bash
# System
uname -a
lsb_release -a
hostnamectl
timedatectl
reboot

# Resources
htop
free -h
df -h
sudo du -sh /var/* | sort -h

# Network
ip addr
ip route
ss -tlnp
dig +short gitlab.<YOUR_DOMAIN>
curl -4 ifconfig.me

# Logs
journalctl -u k3s -f
journalctl -xe
sudo tail -f /var/log/syslog

# Firewall
sudo ufw status verbose
sudo fail2ban-client status sshd

# Packages
sudo apt update
sudo apt full-upgrade -y
apt search <package>
```

---

## kubectl cheat sheet

```bash
# Context
kubectl config current-context
kubectl config get-contexts

# Nodes and namespaces
kubectl get nodes -o wide
kubectl get ns

# Workloads
kubectl get all -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs -f <pod> -n <namespace>
kubectl exec -it <pod> -n <namespace> -- /bin/sh

# Deploy
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl rollout restart deployment/<name> -n <namespace>
kubectl scale deployment/<name> --replicas=2 -n <namespace>

# Debug
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl run tmp --rm -it --image=busybox -- /bin/sh

# Secrets and config
kubectl get secrets -n <namespace>
kubectl create secret generic <name> --from-literal=key=value -n <namespace>

# ArgoCD
kubectl get applications -n argocd
argocd app list
argocd app sync <app>
argocd app diff <app>
```

---

## Docker cheat sheet

```bash
# Info
docker version
docker info
docker ps -a
docker images

# Run
docker run -d --name web -p 8080:80 nginx:alpine
docker stop web && docker rm web

# Build
docker build -t myapp:latest .
docker tag myapp:latest registry.<YOUR_DOMAIN>/lab/myapp:latest
docker push registry.<YOUR_DOMAIN>/lab/myapp:latest

# Login
docker login registry.<YOUR_DOMAIN>

# Cleanup
docker container prune
docker image prune -a
docker system df

# Logs and inspect
docker logs -f <container>
docker inspect <container>
docker exec -it <container> /bin/sh

# Compose
docker compose up -d
docker compose down
docker compose logs -f
```

---

## Git cheat sheet

```bash
# Config
git config --global --list
git config user.name "Name"
git config user.email "you@example.com"

# Daily
git status
git diff
git add .
git commit -m "message"
git push origin main
git pull origin main

# Branches
git branch
git checkout -b feature/x
git merge main

# Remote
git remote -v
git clone git@gitlab.<YOUR_DOMAIN>:lab/gitops.git

# Undo
git restore <file>
git reset --soft HEAD~1   # undo last commit, keep changes

# Log
git log --oneline -10
git show <commit>
```

---

## Placeholder reference

| Placeholder | Example |
|-------------|---------|
| `<YOUR_DOMAIN>` | `lab.example.com` |
| `<SERVER_IP>` | `192.168.1.100` |
| `<PUBLIC_IP>` | Digi WAN IP |
| `<CLOUDFLARE_API_TOKEN>` | From Cloudflare dashboard |
| `<REGISTRY_USER>` | GitLab deploy token user |
| `<REGISTRY_TOKEN>` | GitLab deploy token |

---

## External links

- [Ubuntu Server docs](https://ubuntu.com/server/docs)
- [K3s documentation](https://docs.k3s.io/)
- [GitLab CE docs](https://docs.gitlab.com/ee/install/)
- [ArgoCD docs](https://argo-cd.readthedocs.io/)
- [Traefik docs](https://doc.traefik.io/traefik/)
- [cert-manager docs](https://cert-manager.io/docs/)
- [Cloudflare DNS](https://developers.cloudflare.com/dns/)

---

## Document index

Return to [docs/README.md](./README.md) for the full chapter list.
