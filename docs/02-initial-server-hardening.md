# Chapter 02 — Initial Server Hardening

## Overview

Secure the fresh Ubuntu install: dedicated admin user (if needed), SSH keys, disable password login, firewall, Fail2Ban, and unattended security updates.

## Prerequisites

- [Chapter 01 — Ubuntu 24 Installation](./01-ubuntu-24-installation.md)

## Goals

- [ ] Admin user with sudo access
- [ ] SSH key authentication only
- [ ] UFW firewall enabled with minimal open ports
- [ ] Fail2Ban protecting SSH
- [ ] Automatic security updates enabled

## Steps

### 1. Create admin user (optional)

If you installed with `labadmin`, skip this. Otherwise:

```bash
sudo adduser labadmin
sudo usermod -aG sudo labadmin
```

### 2. SSH keys from your workstation

On your **local machine** (not the server):

```bash
ssh-keygen -t ed25519 -C "labadmin@minipc" -f ~/.ssh/id_ed25519_lab
```

Copy the public key to the server:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub labadmin@<SERVER_IP>
```

Test key login:

```bash
ssh -i ~/.ssh/id_ed25519_lab labadmin@<SERVER_IP>
```

### 3. Harden SSH

On the server:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

Set or add:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers labadmin
```

Validate and restart:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

**Keep your current SSH session open** while testing a new connection in another terminal.

### 4. Configure UFW

Allow SSH first, then enable:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
# HTTP/HTTPS for Traefik — add now or in chapter 14
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

K3s note: when K3s is installed (chapter 11), you may need to allow internal cluster ports between nodes. For single-node, defaults are sufficient.

### 5. Install and configure Fail2Ban

```bash
sudo apt install -y fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local` — ensure `[sshd]` is enabled:

```ini
[sshd]
enabled = true
port = ssh
maxretry = 5
bantime = 3600
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### 6. Automatic security updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
# Select "Yes"
```

Optional — verify config:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Expected:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

### 7. Set timezone

```bash
sudo timedatectl set-timezone Europe/Bucharest
timedatectl
```

Adjust timezone if needed.

## Verify

```bash
# Password auth disabled (should fail from client without key)
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no labadmin@<SERVER_IP>

# UFW active
sudo ufw status

# Fail2Ban running
sudo fail2ban-client status

# Unattended upgrades
systemctl is-active unattended-upgrades
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Locked out of SSH | Use console/IPMI; restore `sshd_config.bak` |
| UFW blocks GitLab/K3s later | Add specific rules; document in relevant chapter |

## Next

→ [Chapter 03 — Network Configuration](./03-network-configuration.md)
