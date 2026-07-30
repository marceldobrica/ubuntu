# Chapter 05 — Cloudflare

## Overview

Register or transfer a domain to Cloudflare, delegate DNS, create subdomain records for GitLab, ArgoCD, and application sites, and choose SSL and access strategy (proxy vs DNS-only, optional Tunnel).

## Prerequisites

- [Chapter 04 — Digi Internet](./04-digi-internet.md)
- Cloudflare account

## Goals

- [ ] Domain active on Cloudflare
- [ ] Nameservers updated at registrar
- [ ] DNS records for lab subdomains
- [ ] SSL mode chosen
- [ ] Optional: Cloudflare Tunnel for CGNAT

## Steps

### 1. Add domain to Cloudflare

1. Log in to [dash.cloudflare.com](https://dash.cloudflare.com)
2. **Add a site** → enter `<YOUR_DOMAIN>`
3. Select Free plan (sufficient for lab)
4. Cloudflare scans existing DNS — review records

### 2. Update nameservers

At your registrar (or if purchased on Cloudflare, this is automatic), set nameservers to Cloudflare's assigned pair, e.g.:

```
ada.ns.cloudflare.com
bob.ns.cloudflare.com
```

Wait for propagation (minutes to 48 hours):

```bash
dig NS <YOUR_DOMAIN> +short
```

### 3. DNS records (initial)

Replace `<PUBLIC_IP>` with your home public IP from chapter 04.

| Type | Name | Content | Proxy | Notes |
|------|------|---------|-------|-------|
| A | `@` | `<PUBLIC_IP>` | DNS only | Optional root |
| A | `gitlab` | `<PUBLIC_IP>` | DNS only* | GitLab |
| A | `argo` | `<PUBLIC_IP>` | DNS only* | ArgoCD |
| A | `drupal` | `<PUBLIC_IP>` | DNS only* | Sample app |
| A | `wp` | `<PUBLIC_IP>` | DNS only* | WordPress |
| A | `symfony` | `<PUBLIC_IP>` | DNS only* | Symfony |
| A | `grafana` | `<PUBLIC_IP>` | DNS only* | Monitoring |

\* **DNS only (grey cloud)** recommended when Traefik terminates Let's Encrypt certs on the origin. Orange cloud (proxied) works but requires Full (strict) SSL and valid origin certs — configure after chapter 15.

### 4. SSL/TLS mode

In Cloudflare → SSL/TLS:

| Phase | Mode | When |
|-------|------|------|
| Initial setup (before origin certs) | **Flexible** or DNS-only records | Temporary |
| After cert-manager (chapter 15) | **Full (strict)** | Production-like |

For Let's Encrypt on Traefik with DNS-01 via Cloudflare, origin certificates will be valid — use **Full (strict)** with proxy enabled if desired.

### 5. Cloudflare API token (for cert-manager)

Create token: **My Profile → API Tokens → Create Token**

Template: **Edit zone DNS**

Permissions:

- Zone → DNS → Edit
- Zone → Zone → Read

Scope: specific zone `<YOUR_DOMAIN>`

Save token securely — used in chapter 15:

```
<CLOUDFLARE_API_TOKEN>
```

### 6. Optional — Cloudflare Tunnel (no port forward)

If CGNAT or you prefer no open ports:

```bash
# On MiniPC — install cloudflared (example)
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
cloudflared tunnel login
cloudflared tunnel create lab-tunnel
```

Configure ingress routes in `~/.cloudflared/config.yml` after Traefik is running (chapter 14). Document routes per subdomain.

### 7. Dynamic DNS update script (optional)

If public IP changes and you use A records:

```bash
#!/bin/bash
# ~/bin/update-cloudflare-dns.sh — skeleton; fill ZONE_ID, RECORD_ID, TOKEN
CURRENT_IP=$(curl -4 -s ifconfig.me)
# curl PATCH to Cloudflare API...
```

Run via cron every 10 minutes.

## Verify

```bash
dig gitlab.<YOUR_DOMAIN> +short
dig argo.<YOUR_DOMAIN> +short
curl -I https://gitlab.<YOUR_DOMAIN>
# May fail until GitLab/Traefik installed — DNS should resolve
```

Checklist:

- [ ] Domain status **Active** in Cloudflare
- [ ] Subdomain A records or Tunnel routes defined
- [ ] API token created for cert-manager
- [ ] SSL mode documented

## Troubleshooting

| Problem | Fix |
|---------|-----|
| DNS not resolving | Wait propagation; verify nameservers |
| SSL errors with orange cloud | Switch to DNS only until origin cert ready |
| 522/521 from Cloudflare | Origin unreachable — check port forward/Tunnel |

## Next

→ [Chapter 06 — Docker](./06-docker.md)
