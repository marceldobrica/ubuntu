# Chapter 21 — Backup

## Overview

Backup GitLab, container registry data, and Kubernetes PVCs. Run periodic restore tests so backups are trusted before production-like use.

## Prerequisites

- [Chapter 08 — GitLab CE](./08-gitlab-ce.md)
- [Chapter 11 — Kubernetes (K3s)](./11-kubernetes-k3s.md)
- Application platforms deployed (chapters 16–18)

## Goals

- [ ] GitLab backup scheduled
- [ ] Registry / image retention policy
- [ ] PVC snapshot or volume backup
- [ ] Restore procedure documented and tested once

## Steps

### 1. GitLab backup

GitLab Omnibus backup includes repos, DB, registry (if configured), uploads:

```bash
sudo gitlab-backup create
```

Backups stored in `/var/opt/gitlab/backups/` by default.

Configure in `/etc/gitlab/gitlab.rb`:

```ruby
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"
gitlab_rails['backup_keep_time'] = 604800  # 7 days
```

Cron as root:

```bash
sudo crontab -e
# Daily at 2 AM
0 2 * * * /opt/gitlab/bin/gitlab-backup create CRON=1
```

Copy backups off-site (external disk, another machine):

```bash
rsync -avz /var/opt/gitlab/backups/ labadmin@<BACKUP_HOST>:/backups/gitlab/
```

### 2. GitLab config backup

```bash
sudo cp /etc/gitlab/gitlab.rb /var/opt/gitlab/backups/gitlab.rb.$(date +%F)
sudo cp /etc/gitlab/gitlab-secrets.json /var/opt/gitlab/backups/
```

### 3. Container registry

Registry data lives under GitLab backup if integrated. Verify:

```bash
sudo gitlab-ctl show-config | grep registry
```

Additional: export critical images to tarball periodically:

```bash
docker save registry.<YOUR_DOMAIN>/lab/symfony-app:latest -o ~/backups/symfony-app-latest.tar
```

### 4. Kubernetes etcd (K3s)

K3s etcd snapshot:

```bash
sudo k3s etcd-snapshot save --name manual-$(date +%F)
```

Default location: `/var/lib/rancher/k3s/server/db/snapshots/`

Automate:

```bash
sudo crontab -e
0 3 * * * /usr/local/bin/k3s etcd-snapshot save --name scheduled-$(date +\%F)
```

### 5. PVC backup (Velero or restic)

**Velero** (lab-friendly):

```bash
# Install Velero CLI + server component
# Backup namespace drupal, wordpress, symfony to local storage or MinIO
velero backup create apps-backup --include-namespaces drupal,wordpress,symfony
```

**Simple manual PVC backup** (K3s local-path):

```bash
# Identify pod using PVC
kubectl get pvc -A
# Copy data via kubectl cp or temporary mount job
```

### 6. GitOps repository

GitLab hosts the source of truth — covered by GitLab backup. Additionally mirror to secondary remote optionally.

### 7. Restore test procedure

**GitLab (test quarterly on spare VM or staging):**

```bash
sudo gitlab-ctl stop puma
sudo gitlab-ctl stop sidekiq
sudo gitlab-backup restore BACKUP=<timestamp>
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
```

**K3s etcd:**

```bash
sudo k3s server --cluster-reset --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<snapshot>
```

**Document actual timestamps and paths when testing.**

## Verify

```bash
sudo ls -lh /var/opt/gitlab/backups/
sudo ls -lh /var/lib/rancher/k3s/server/db/snapshots/
velero backup get  # if Velero installed
```

Checklist:

- [ ] Last GitLab backup < 24h old
- [ ] etcd snapshot exists
- [ ] Restore steps written in runbook (this chapter)
- [ ] One restore drill completed (mark Tested in docs index)

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Backup too large | Exclude artifacts; registry cleanup |
| Restore version mismatch | Match GitLab version to backup |
| PVC restore empty | Verify backup included correct volume |

## Next

→ [Chapter 22 — Security](./22-security.md)
