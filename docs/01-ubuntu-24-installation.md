# Chapter 01 — Ubuntu 24 Installation

## Overview

Install Ubuntu Server 24.04 LTS on the MiniPC with UEFI, SSH server enabled, and a minimal base suitable for Docker and K3s.

## Prerequisites

- [Chapter 00 — Introduction](./00-introduction.md)
- USB drive (8 GB+)
- Ethernet connection recommended for first boot

## Goals

- [x] Ubuntu Server 24.04 LTS installed
- [x] UEFI boot working
- [x] SSH server enabled
- [x] System fully updated

## Steps

### 1. Download Ubuntu Server

Download the latest **24.04 LTS** ISO from [ubuntu.com/download/server](https://ubuntu.com/download/server).

Verify checksum (optional):

```bash
sha256sum ubuntu-24.04.*-live-server-amd64.iso
```

### 2. Create bootable USB

On Linux:

```bash
sudo apt install usb-creator-gtk
# Or use balenaEtcher: https://etcher.balena.io
```

Write the ISO to the USB drive. **All data on the USB will be erased.**

### 3. BIOS / UEFI settings

Before booting from USB, configure firmware:

| Setting | Recommendation |
|---------|----------------|
| Boot mode | UEFI (disable Legacy/CSM if possible) |
| Secure Boot | Disabled (simplifies Docker/K3s drivers initially) |
| Virtualization (VT-x/AMD-V) | Enabled |
| Boot order | USB first, then NVMe |

**Old UEFI note:** If the installer hangs at boot, edit GRUB (press `e`) and add `nolapic` to the linux line. Some older MiniPC boards need this.

### 4. Install Ubuntu Server

Boot from USB and follow the installer:

1. **Language / keyboard** — your preference
2. **Installation type** — Ubuntu Server (minimized if offered; full server is fine)
3. **Network** — DHCP is OK for now; static IP comes in chapter 03
4. **Proxy** — leave empty unless required
5. **Mirror** — default
6. **Storage** — use entire disk (or custom LVM if you prefer snapshots later)
7. **Profile setup**
   - Hostname: e.g. `lab-node-01`
   - Username: e.g. `labadmin` (avoid generic `admin`)
   - Password: strong; you will move to SSH keys in chapter 02
8. **SSH** — **Install OpenSSH server** ✓
9. **Snaps** — optional; none required for this lab
10. Reboot and remove USB

### 5. First login and update

SSH from your workstation (replace IP):

```bash
ssh labadmin@<SERVER_IP>
```

Update the system:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

### 6. Install useful base packages

```bash
sudo apt install -y \
  curl \
  wget \
  git \
  vim \
  htop \
  net-tools \
  ca-certificates \
  gnupg \
  lsb-release \
  software-properties-common
```

## Verify

```bash
# OS version
lsb_release -a

# UEFI boot (should show EFI directory)
[ -d /sys/firmware/efi ] && echo "UEFI boot: OK" || echo "Legacy boot"

# SSH running
systemctl is-active ssh

# Kernel and resources
uname -r
nproc
free -h
df -h /
```

Expected: Ubuntu 24.04, SSH `active`, sufficient disk free.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Installer freezes | Add `nolapic` at GRUB; update BIOS |
| No network in installer | Use Ethernet; check cable/VLAN |
| SSH refused after install | Log in locally; `sudo systemctl enable --now ssh` |

## Next

→ [Chapter 02 — Initial Server Hardening](./02-initial-server-hardening.md)
