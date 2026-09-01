# Chapter 08 — GitLab CE

## Overview

Install GitLab Community Edition with HTTPS at `gitlab.<YOUR_DOMAIN>`, bundled PostgreSQL, and the container registry enabled.

## Prerequisites

- [Chapter 07 — Git](./07-git.md)
- DNS `gitlab.<YOUR_DOMAIN>` pointing to your server
- Minimum 8 GB RAM recommended

## Goals

- [ ] GitLab CE running and reachable via HTTPS
- [ ] Root password captured
- [ ] 2FA enabled for root and administrators
- [ ] Container registry enabled
- [ ] Backup strategy noted (chapter 21)

## Steps

### 1. Install dependencies

```bash
sudo apt install -y curl openssh-server ca-certificates tzdata perl
```

### 2. Add GitLab package repository

```bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

### 3. Install GitLab CE

Set external URL before install:

```bash
sudo EXTERNAL_URL="https://gitlab.<YOUR_DOMAIN>" apt install -y gitlab-ce
```

Initial configure may take several minutes.

### 4. Initial configuration

Edit `/etc/gitlab/gitlab.rb` for key settings:

```bash
sudo nano /etc/gitlab/gitlab.rb
```

Important lines (uncomment/modify as needed):

```ruby
external_url 'https://gitlab.<YOUR_DOMAIN>'

# Registry on same host
registry_external_url 'https://registry.<YOUR_DOMAIN>'

# Let GitLab obtain LE cert (alternative to Traefik — pick one approach)
# nginx['redirect_http_to_https'] = true
# letsencrypt['enable'] = true
# letsencrypt['contact_emails'] = ['you@example.com']
```

**TLS note:** You can use GitLab's built-in Let's Encrypt **or** terminate TLS via Traefik (chapter 14). For a unified wildcard cert strategy, prefer Traefik/cert-manager and set GitLab to HTTP behind ingress — document your choice here.

Apply changes:

```bash
sudo gitlab-ctl reconfigure
```

### 5. Get initial root password

```bash
sudo cat /etc/gitlab/initial_root_password
```

Log in at `https://gitlab.<YOUR_DOMAIN>` as `root`, change password immediately.

### 6. Enable two-factor authentication

Enable 2FA immediately for `root`:

1. Open **Edit profile → Access → Password and authentication**.
2. Under **Two-factor authentication**, choose **Enable two-factor
   authentication**.
3. Scan the QR code with an authenticator app and enter the six-digit code.
4. Store the recovery codes in the password manager.

Create a separate named administrator account and enable 2FA for it as well.
Do not use `root` for routine administration.

After confirming that recovery codes and the separate administrator account
work, require 2FA for all users under **Admin Area → Settings → General →
Sign-in restrictions → Two-factor authentication**. Use a grace period only if
needed for onboarding.

### 7. Add SSH public key

As root (or new admin user):

1. **Preferences → SSH Keys**
2. Paste `~/.ssh/id_ed25519_gitlab.pub` from chapter 07

### 8. Enable Container Registry

In `gitlab.rb` (often enabled by default with `registry_external_url`):

```ruby
registry['enable'] = true
registry_nginx['listen_port'] = 5050
```

Reconfigure:

```bash
sudo gitlab-ctl reconfigure
```

Add DNS A record for `registry.<YOUR_DOMAIN>` if separate hostname.

### 9. Create groups and projects (skeleton)

Via UI:

- Group: `lab`
- Projects: `gitops`, `drupal-site`, `wordpress-site`, `symfony-app`

### 10. Backup (preview)

GitLab backup command (full backup in chapter 21):

```bash
sudo gitlab-backup create
```

## Verify

```bash
sudo gitlab-ctl status
curl -Ik https://gitlab.<YOUR_DOMAIN>
ssh -T git@gitlab.<YOUR_DOMAIN>
```

UI checklist:

- [ ] Login works
- [ ] 2FA enabled for root and administrators
- [ ] SSH clone works: `git clone git@gitlab.<YOUR_DOMAIN>:lab/gitops.git`
- [ ] Registry UI visible under Deploy → Container Registry

## Troubleshooting

| Problem | Fix |
|---------|-----|
| 502 during startup | Wait; check `sudo gitlab-ctl tail` |
| High memory | Reduce Prometheus in `gitlab.rb` for lab: `prometheus_monitoring['enable'] = false` |
| Cert errors | Align with Cloudflare SSL mode (chapter 05) |

## Next

→ [Chapter 09 — GitLab Runner](./09-gitlab-runner.md)
