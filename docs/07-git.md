# Chapter 07 — Git

## Overview

Configure Git for commits from the MiniPC and set up SSH keys for GitLab (chapter 08) and the GitOps repository.

## Prerequisites

- [Chapter 06 — Docker](./06-docker.md)

## Goals

- [ ] Global Git identity configured
- [ ] SSH key for GitLab
- [ ] SSH config for multiple hosts (optional)

## Steps

### 1. Install Git (if not present)

```bash
sudo apt install -y git
git --version
```

### 2. Global configuration

```bash
git config --global user.name "Lab Admin"
git config --global user.email "labadmin@<YOUR_DOMAIN>"
git config --global init.defaultBranch main
git config --global pull.rebase false
git config --global core.editor vim
```

### 3. Generate SSH key for GitLab

```bash
ssh-keygen -t ed25519 -C "gitlab@lab-node-01" -f ~/.ssh/id_ed25519_gitlab
```

Add to SSH agent:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_gitlab
```

### 4. SSH config (recommended)

```bash
nano ~/.ssh/config
```

```
Host gitlab.<YOUR_DOMAIN>
  HostName gitlab.<YOUR_DOMAIN>
  User git
  IdentityFile ~/.ssh/id_ed25519_gitlab
  IdentitiesOnly yes
```

Permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config ~/.ssh/id_ed25519_gitlab
chmod 644 ~/.ssh/id_ed25519_gitlab.pub
```

### 5. Display public key

Copy output for GitLab (chapter 08):

```bash
cat ~/.ssh/id_ed25519_gitlab.pub
```

## Verify

```bash
git config --list --global
ssh -T git@gitlab.<YOUR_DOMAIN>
# Expect welcome message after GitLab is installed
```

Until GitLab exists, verify key files only:

```bash
ls -la ~/.ssh/id_ed25519_gitlab*
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Permission denied (publickey) | Add pubkey to GitLab user settings |
| Host key verification failed | `ssh-keyscan gitlab.<YOUR_DOMAIN> >> ~/.ssh/known_hosts` |

## Next

→ [Chapter 08 — GitLab CE](./08-gitlab-ce.md)
