# Chapter 04 — Digi Internet

## Overview

Configure Digi (RCS & RDS) home internet for external access: public IP awareness, dynamic DNS or update strategy, router port forwarding to the MiniPC.

## Prerequisites

- [Chapter 03 — Network Configuration](./03-network-configuration.md)
- Digi router admin access

## Goals

- [ ] Know whether you have public or CGNAT IP
- [ ] Port forwarding 80/443 (and optionally 22) to MiniPC
- [ ] Strategy for dynamic IP changes documented

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

### 3. Port forwarding

Forward to your MiniPC static IP (`192.168.1.100` in chapter 03):

| External port | Internal IP | Internal port | Protocol | Purpose |
|---------------|-------------|---------------|----------|---------|
| 80 | 192.168.1.100 | 80 | TCP | HTTP (Let's Encrypt, redirects) |
| 443 | 192.168.1.100 | 443 | TCP | HTTPS (Traefik) |
| 22 | 192.168.1.100 | 22 | TCP | SSH (optional; restrict by IP if exposed) |

**Security:** Prefer SSH via VPN or Cloudflare Tunnel rather than exposing port 22 publicly.

### 4. Dynamic public IP handling

Digi residential IPs often change after reboot or periodically.

Options:

**A. Cloudflare API update script (if using DNS-only A record)**

Cron job on MiniPC to update Cloudflare when IP changes — see chapter 05.

**B. Cloudflare Tunnel (recommended if CGNAT or unstable IP)**

No port forwarding required; outbound tunnel from MiniPC.

**C. Digi fixed IP (paid option)**

Simplest for production-like lab; ask Digi for business/static IP if available.

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

- [ ] Public IP documented
- [ ] Port forward rules saved in router
- [ ] External port check succeeds (or Tunnel plan noted for chapter 05)
- [ ] CGNAT status known

## Troubleshooting

| Problem | Fix |
|---------|-----|
| CGNAT — ports unreachable | Use Cloudflare Tunnel; skip port forward |
| IP changes break DNS | Automate Cloudflare updates or use Tunnel |
| Double NAT | Put Digi router in bridge mode or forward from upstream router |

## Next

→ [Chapter 05 — Cloudflare](./05-cloudflare.md)
