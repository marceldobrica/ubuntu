# Chapter 24 — Troubleshooting

## Overview

Common problems and recovery commands for the lab stack: Linux host, Docker, GitLab, K3s, ArgoCD, Traefik, and applications.

## Prerequisites

All prior chapters — use as reference during incidents.

## Common problems

### Cannot reach site from internet

1. Check public IP: `curl -4 ifconfig.me`
2. Cloudflare DNS points to correct IP
3. Router port forward 80/443 → MiniPC
4. UFW allows 80/443: `sudo ufw status`
5. Traefik service: `kubectl get svc -n traefik`
6. If CGNAT: use Cloudflare Tunnel

### Certificate not issued

```bash
kubectl describe certificate -A
kubectl get challenges -A
kubectl logs -n cert-manager -l app=cert-manager -f
```

Verify Cloudflare token permissions and DNS propagation.

### ArgoCD OutOfSync / Sync failed

```bash
argocd app get <app> --refresh
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

Fix manifest errors in GitOps repo; commit and re-sync.

### GitLab 502 / slow

```bash
sudo gitlab-ctl status
sudo gitlab-ctl tail
free -h
```

Reduce memory: disable bundled Prometheus, reduce unicorn/puma workers in `gitlab.rb`.

### GitLab hostname returns 406 while the server is up or down

If `curl` returns the same 406 response regardless of GitLab's state, first
identify which layer generated it:

```bash
curl -sS -D - -o /dev/null https://gitlab.<YOUR_DOMAIN>
curl -sS -D - -o /dev/null http://gitlab.<YOUR_DOMAIN>
dig gitlab.<YOUR_DOMAIN> A +short
dig gitlab.<YOUR_DOMAIN> CNAME +short
```

Headers such as `server: cloudflare`, a Cloudflare Ray ID, or a fixed
redirect target indicate that Cloudflare is responding before the origin.
Check the hostname's DNS record, Tunnel published application, Redirect Rules,
Workers, and Access application. Remove any stale duplicate DNS record and
correct invalid redirect targets such as `https://255.255.255.255/`.

Only after the public route reaches the host should you test the origin
directly:

```bash
sudo ss -tlnp | grep -E ':(80|443)\b'
sudo gitlab-ctl status
curl -I http://127.0.0.1
```

If the public request has no Cloudflare headers and the local request also
returns 406, inspect the GitLab nginx logs and any ModSecurity or reverse-proxy
configuration on the host:

```bash
sudo gitlab-ctl tail nginx
sudo journalctl -u nginx --no-pager -n 100
```

### Runner jobs pending

```bash
sudo gitlab-runner verify
sudo gitlab-runner list
```

Check tags, runner paused state, Docker socket.

### Pod CrashLoopBackOff

```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Database connection errors

Verify Service name, secrets, and DB pod ready:

```bash
kubectl get pods -n drupal -l app=mariadb
kubectl exec -it -n drupal deploy/mariadb -- mysql -u drupal -p
```

## kubectl commands

```bash
# Cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running

# Resource usage (needs metrics-server)
kubectl top nodes
kubectl top pods -A

# Logs
kubectl logs -f -n <ns> -l app=<app>

# Shell into pod
kubectl exec -it -n <ns> deploy/<name> -- /bin/sh

# Restart deployment
kubectl rollout restart deployment/<name> -n <ns>
kubectl rollout status deployment/<name> -n <ns>

# Force ArgoCD hard refresh
kubectl patch application <app> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

## Docker commands

```bash
docker ps -a
docker logs <container>
docker system df
docker system prune -a   # careful — removes unused images

# GitLab specific
sudo gitlab-ctl restart
sudo gitlab-ctl reconfigure
```

## GitLab commands

```bash
sudo gitlab-ctl status
sudo gitlab-ctl tail nginx
sudo gitlab-ctl tail postgresql
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-backup create
```

## K3s commands

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -f
sudo k3s kubectl get nodes
sudo k3s etcd-snapshot save --name emergency-$(date +%F)
```

## Recovery procedures

### Reset failed ArgoCD app

```bash
argocd app delete <app> --cascade
kubectl apply -f apps/<app>.yaml
argocd app sync <app>
```

### Restore GitLab from backup

See [Chapter 21 — Backup](./21-backup.md).

### K3s cluster reset (destructive)

```bash
sudo /usr/local/bin/k3s-uninstall.sh
# Reinstall from chapter 11; restore from etcd snapshot if available
```

### Traefik not routing

```bash
kubectl logs -n traefik -l app.kubernetes.io/name=traefik
kubectl get ingressroute -A
```

## Next

→ [Chapter 25 — Appendix](./25-appendix.md)
