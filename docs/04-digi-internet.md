# Chapter 04 — Digi Internet

## Overview

Configure Digi (RCS & RDS) home internet for external access. This lab has a
verified fixed public IP, and the Cloudflare Tunnel has also been tested
successfully. The fixed IP supports direct access when required, while the
Tunnel remains useful when inbound ports should not be exposed.

## Prerequisites

- [Chapter 03 — Network Configuration](./03-network-configuration.md)
- Digi router admin access

## Goals

- [x] Fixed public IP verified
- [x] Cloudflare Tunnel tested
- [x] Access strategy documented

## Steps

### 1. Check public IP vs WAN IP

On the MiniPC:

```bash
curl -4 ifconfig.me
```

Compare with the WAN IP shown in the Digi router admin panel.

| Situation | Meaning |
|-----------|---------|
| Same IP | You likely have a public IP — port forwarding will work |
| Different IP (carrier-grade NAT) | Direct inbound may not work — use **Cloudflare Tunnel** (chapter 05) |

### 2. Digi router access

Typical gateway: `192.168.1.1` or `192.168.0.1`

Log in with credentials from the router sticker or Digi account.

### 3. Optional port forwarding

Port forwarding is optional when using Cloudflare Tunnel. If you choose direct
inbound access, forward to your MiniPC static IP (`192.168.1.100` in chapter
03):

| External port | Internal IP | Internal port | Protocol | Purpose |
|---------------|-------------|---------------|----------|---------|
| 80 | 192.168.1.100 | 80 | TCP | HTTP (Let's Encrypt, redirects) |
| 443 | 192.168.1.100 | 443 | TCP | HTTPS (Traefik) |
| 22 | 192.168.1.100 | 22 | TCP | SSH (optional; restrict by IP if exposed) |

**Security:** Prefer SSH via VPN or Cloudflare Tunnel rather than exposing port 22 publicly.

### 4. Public IP and access strategy

The Digi connection has a verified fixed public IP. Direct public-IP access is
therefore available when the required router port-forwarding rules are
configured. Cloudflare Tunnel was also tested successfully and provides an
alternative that does not require inbound ports.

Options:

**A. Cloudflare API update script (if using DNS-only A record)**

This is not needed while the fixed IP remains unchanged. The update script in
chapter 05 has **not been tested yet** and should be tested after the remaining
lab setup is finished.

**B. Cloudflare Tunnel**

The outbound tunnel from the MiniPC works without inbound port forwarding. It
was tested successfully; see chapter 05.

Document your current public IP:

```bash
echo "$(date): $(curl -4 -s ifconfig.me)" >> ~/public-ip.log
```

### 5. Test inbound connectivity

From **outside** your home network (mobile data):

```bash
curl -I http://<PUBLIC_IP>
# Expect connection (may 404 until Traefik is installed)
nc -zv <PUBLIC_IP> 443
```

## Verify

- [x] Fixed public IP verified
- [x] Cloudflare Tunnel tested
- [ ] Cloudflare API update script tested (follow-up after the lab setup)

## Troubleshooting

| Problem | Fix |
|---------|-----|
| CGNAT — ports unreachable | Use Cloudflare Tunnel; skip port forward |
| IP changes break DNS | Automate Cloudflare updates or use Tunnel |
| Double NAT | Put Digi router in bridge mode or forward from upstream router |

## Next

→ [Chapter 05 — Cloudflare](./05-cloudflare.md)
