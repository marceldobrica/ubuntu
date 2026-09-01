# Chapter 03 — Network Configuration

## Overview

Configure a static IP, hostname, DNS, and timezone so services and DNS records remain stable across reboots.

## Prerequisites

- [Chapter 02 — Initial Server Hardening](./02-initial-server-hardening.md)
- Know your LAN subnet, gateway, and desired static IP

## Goals

- [x] Static IP via Netplan
- [x] Hostname set
- [x] DNS resolvers configured
- [x] Network persists after reboot

## Steps

### 1. Identify network interface

```bash
ip link show
ip route show default
```

Note the interface name (e.g. `enp2s0`, `eth0`).

### 2. Configure Netplan

Ubuntu 24 uses Netplan. Example for interface `enp2s0`:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp2s0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Replace:

- `enp2s0` — your interface
- `192.168.1.100` — chosen static IP
- `192.168.1.1` — your router/gateway

Secure the configuration file before applying it:

```bash
sudo chown root:root /etc/netplan/00-installer-config.yaml
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

If `/etc/netplan/50-cloud-init.yaml` also exists and contains
`dhcp4: true`, cloud-init will add a DHCP address alongside the static
address. Disable cloud-init network configuration so it does not recreate
the conflicting file:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

Add:

```yaml
network: {config: disabled}
```

Then remove the generated Netplan file and regenerate the configuration:

```bash
sudo rm /etc/netplan/50-cloud-init.yaml
sudo netplan generate
```

Apply:

```bash
sudo netplan try
# Confirm within timeout, or it reverts
sudo netplan apply
```

### 3. Set hostname

```bash
sudo hostnamectl set-hostname lab-node-01
```

Edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

```
127.0.0.1 localhost
127.0.1.1 lab-node-01
192.168.1.100 lab-node-01
```

### 4. Verify DNS resolution

```bash
resolvectl status
ping -c 2 github.com
ping -c 2 gitlab.com
```

## Verify

```bash
hostname
hostname -f
ip addr show
ip route
resolvectl query google.com
```

After reboot:

```bash
sudo reboot
# Reconnect and confirm IP unchanged
ip -4 addr show
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `netplan try` fails | Check YAML indentation (spaces, not tabs) |
| `Permissions for /etc/netplan/*.yaml are too open` | Run `sudo chown root:root /etc/netplan/00-installer-config.yaml` and `sudo chmod 600 /etc/netplan/00-installer-config.yaml` |
| Two IPv4 addresses appear | Check for `/etc/netplan/50-cloud-init.yaml` with `dhcp4: true`; disable cloud-init network configuration as shown above, remove the file, and run `sudo netplan generate` |
| No internet | Verify gateway IP; test router ping |
| Wrong interface name | Use `ip link`; update YAML |

## Next

→ [Chapter 04 — Digi Internet](./04-digi-internet.md)
