# Chapter 05 — Cloudflare

## Overview

Register or transfer a domain to Cloudflare, delegate DNS, create subdomain
records for GitLab, ArgoCD, and application sites, and choose SSL and access
strategy. When using a Cloudflare Tunnel, DNS points to the tunnel rather than
to the Digi public IP.

## Prerequisites

- [Chapter 04 — Digi Internet](./04-digi-internet.md)
- Cloudflare account

## Goals

- [x] Domain active on Cloudflare
- [x] Nameservers updated at registrar
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

Choose **one** of these approaches for each hostname. Do not create an `A`
record pointing at the Digi public IP for a hostname served through a Tunnel.

#### Cloudflare Tunnel (recommended)

Create the Tunnel first under **Zero Trust → Networks → Tunnels**. Install and
run its connector on the MiniPC, then add a **Published application** route for every
service:

| Hostname | Service URL on the MiniPC |
|----------|----------------------------|
| `gitlab.<YOUR_DOMAIN>` | `http://127.0.0.1:80` or the local GitLab/Traefik URL |
| `argo.<YOUR_DOMAIN>` | `http://127.0.0.1:<ARGO_PORT>` |
| `drupal.<YOUR_DOMAIN>` | `http://127.0.0.1:<DRUPAL_PORT>` |
| `grafana.<YOUR_DOMAIN>` | `http://127.0.0.1:<GRAFANA_PORT>` |

Use the actual local listener and port. If Traefik is the single local
entrypoint, point each hostname at Traefik and let the hostname rule select
the service.

When a Published application route is saved, Cloudflare creates the DNS record
automatically:

```text
<hostname>  CNAME  <TUNNEL_UUID>.cfargotunnel.com
```

The record is normally **Proxied** (orange cloud). You do not need to know or
update the changing Digi public IP, and no inbound router port-forward is
required. If a previous `A` record exists with the same name, remove it or
replace it with the Tunnel hostname record.

You can verify the DNS target with:

```bash
dig gitlab.<YOUR_DOMAIN> CNAME +short
```

Do not use a Tunnel CNAME for `vpn`/OpenVPN UDP traffic; Cloudflare's normal
HTTP/HTTPS Tunnel does not proxy arbitrary UDP. Keep `vpn` as a DNS-only `A`
record only when you have a reachable public IP, or use a different remote
access solution.

#### Direct public-IP access (without a Tunnel)

Replace `<PUBLIC_IP>` with your home public IP from chapter 04:

| Type | Name | Content | Proxy | Notes |
|------|------|---------|-------|-------|
| A | `@` | `<PUBLIC_IP>` | DNS only | Optional root |
| A | `vpn` | `<PUBLIC_IP>` | DNS only | OpenVPN endpoint |
| A | `gitlab` | `<PUBLIC_IP>` | DNS only* | GitLab |
| A | `argo` | `<PUBLIC_IP>` | DNS only* | ArgoCD |
| A | `drupal` | `<PUBLIC_IP>` | DNS only* | Sample app |
| A | `wp` | `<PUBLIC_IP>` | DNS only* | WordPress |
| A | `symfony` | `<PUBLIC_IP>` | DNS only* | Symfony |
| A | `grafana` | `<PUBLIC_IP>` | DNS only* | Monitoring |

\* **DNS only (grey cloud)** is recommended when Traefik terminates Let's
Encrypt certificates on the origin. Orange cloud requires Full (strict) SSL
and a valid origin certificate.

### 4. Configure OpenVPN (optional)

Do this only after the server has a public IP and the `vpn.<YOUR_DOMAIN>`
DNS record resolves to it. Cloudflare proxying does not carry OpenVPN UDP
traffic, so keep the `vpn` record **DNS only**. If the connection uses CGNAT
and has no reachable public IP, use Cloudflare Tunnel instead.

OpenVPN can provide remote access to the LAN without exposing SSH publicly.
The examples below use:

- VPN subnet: `10.8.0.0/24`
- OpenVPN port: UDP `1194`
- LAN subnet: `192.168.1.0/24`
- Server LAN interface: `<LAN_INTERFACE>`

Replace these values with the actual network values for this server.

Install OpenVPN and Easy-RSA:

```bash
sudo apt install -y openvpn easy-rsa
sudo install -d -m 700 /etc/openvpn/easy-rsa
sudo cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
sudo -i
cd /etc/openvpn/easy-rsa
```

Create a certificate authority, server certificate, and one client
certificate from the root shell. Use a unique client name for each device:

```bash
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
install -d -m 700 /etc/openvpn/server
openvpn --genkey secret /etc/openvpn/server/ta.key
./easyrsa gen-req laptop nopass
./easyrsa sign-req client laptop
exit
```

Create `/etc/openvpn/server/server.conf`:

```ini
port 1194
proto udp
dev tun
user nobody
group nogroup
persist-key
persist-tun
topology subnet
server 10.8.0.0 255.255.255.0
push "route 192.168.1.0 255.255.255.0"
keepalive 10 120
ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key
dh /etc/openvpn/easy-rsa/pki/dh.pem
tls-crypt /etc/openvpn/server/ta.key
data-ciphers AES-256-GCM:AES-128-GCM
auth SHA256
verb 3
```

Enable IPv4 forwarding:

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-openvpn-forwarding.conf
sudo sysctl --system
```

Allow the VPN endpoint and VPN-to-LAN traffic through UFW. Replace
`<LAN_INTERFACE>` with the interface shown by `ip route`:

```bash
sudo ufw allow 1194/udp
sudo ufw allow from 10.8.0.0/24 to any port 22 proto tcp
sudo ufw route allow in on tun0 out on <LAN_INTERFACE> \
  from 10.8.0.0/24 to 192.168.1.0/24
```

Important: this Ubuntu server is not acting as the LAN router. The Digi router is
already doing NAT/masquerading for the home network. Do not add NAT rules on
Ubuntu unless the server is explicitly configured as the gateway for the LAN.
In this lab, the router forwards UDP `1194` to the server and handles Internet
address translation; the Ubuntu host only needs routing and firewall rules for
VPN traffic, not a MASQUERADE rule.

Restart UFW and OpenVPN, then forward UDP port `1194` from the router to the
server. Do not forward TCP port `22`:

```bash
sudo ufw disable && sudo ufw enable
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server
```

#### Create a client profile

The following commands run on the VPN server. Stage the files in the
`labadmin` home directory so that they can be downloaded without exposing
root's private key permissions:

```bash
sudo install -d -o labadmin -g labadmin -m 700 /home/labadmin/openvpn-client
sudo install -o labadmin -g labadmin -m 600 \
  /etc/openvpn/easy-rsa/pki/private/laptop.key \
  /home/labadmin/openvpn-client/
sudo install -o labadmin -g labadmin -m 644 \
  /etc/openvpn/easy-rsa/pki/issued/laptop.crt \
  /etc/openvpn/easy-rsa/pki/ca.crt \
  /home/labadmin/openvpn-client/
sudo install -o labadmin -g labadmin -m 600 \
  /etc/openvpn/server/ta.key \
  /home/labadmin/openvpn-client/
```

Exit the SSH session, then run the following commands in a terminal on the
client laptop. Replace `<SERVER_IP>` with the server's address:

```bash
mkdir -p ~/openvpn-client
scp labadmin@<SERVER_IP>:/home/labadmin/openvpn-client/laptop.key \
    labadmin@<SERVER_IP>:/home/labadmin/openvpn-client/laptop.crt \
    labadmin@<SERVER_IP>:/home/labadmin/openvpn-client/ca.crt \
    labadmin@<SERVER_IP>:/home/labadmin/openvpn-client/ta.key \
    ~/openvpn-client/
```

`scp` transfers the files over encrypted SSH. After confirming the transfer,
remove the server-side staging directory from the client laptop:

```bash
ssh labadmin@<SERVER_IP> 'sudo rm -rf /home/labadmin/openvpn-client'
```

Create `laptop.ovpn` on the client laptop. Use `vpn.<YOUR_DOMAIN>` as the
OpenVPN endpoint and paste the contents of the copied files into the matching
sections:

```text
client
dev tun
proto udp
remote vpn.<YOUR_DOMAIN> 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
data-ciphers AES-256-GCM:AES-128-GCM
auth SHA256
verb 3

<ca>
Paste the contents of ca.crt here
</ca>
<cert>
Paste the contents of laptop.crt here
</cert>
<key>
Paste the contents of laptop.key here
</key>
<tls-crypt>
Paste the contents of ta.key here
</tls-crypt>
```

Import `laptop.ovpn` into an OpenVPN client, connect, and then SSH to the
server's LAN address:

```bash
ssh labadmin@192.168.1.100
```

Verify the VPN and routing:

```bash
ip addr show tun0
sudo journalctl -u openvpn-server@server --no-pager -n 50
sudo ufw status numbered
```

Create a separate certificate for every client device. Revoke a lost device's
certificate from `/etc/openvpn/easy-rsa` with
`./easyrsa revoke <CLIENT_NAME>`, then regenerate and deploy a CRL.

### 5. SSL/TLS mode

In Cloudflare → SSL/TLS:

| Phase | Mode | When |
|-------|------|------|
| Initial setup (before origin certs) | **Flexible** or DNS-only records | Temporary |
| After cert-manager (chapter 15) | **Full (strict)** | Production-like |

For Let's Encrypt on Traefik with DNS-01 via Cloudflare, origin certificates will be valid — use **Full (strict)** with proxy enabled if desired.

### 6. Cloudflare API token (for cert-manager)

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

### 7. Optional — Cloudflare Tunnel (no port forward)

If CGNAT or you prefer no open ports, use either a remotely managed tunnel or
a locally managed tunnel. Do not mix their configuration methods. A dashboard
tunnel stores routes in Cloudflare; a CLI-created tunnel needs a local
`config.yml`.

#### Remotely managed tunnel (dashboard)

Create the tunnel at **Networking → Tunnels → Create a tunnel**, copy the
connector installation command shown by Cloudflare, and run it on the MiniPC.
Wait until the tunnel is **Healthy**. Add routes under the tunnel's
**Routes → Add route → Published application** tab.

#### Locally managed tunnel (CLI)

```bash
# On MiniPC — install cloudflared (example)
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
cloudflared tunnel login
cloudflared tunnel create lab-tunnel
```

If the tunnel was created with the CLI, create `~/.cloudflared/config.yml`:

```yaml
tunnel: <TUNNEL_UUID>
credentials-file: /home/<USER>/.cloudflared/<TUNNEL_UUID>.json

ingress:
  - hostname: tunnel-test.<YOUR_DOMAIN>
    service: http://127.0.0.1:8080
  - service: http_status:404
```

Then create the DNS CNAME and run the tunnel:

```bash
cloudflared tunnel route dns <TUNNEL_NAME> tunnel-test.<YOUR_DOMAIN>
cloudflared tunnel run <TUNNEL_NAME>
```

The credentials file is created by `cloudflared tunnel create`. Use the
absolute path for the account that runs `cloudflared`. Document one route per
subdomain.

### 8. Dynamic DNS update script (optional)

If public IP changes and you use A records. This script has **not been tested**
yet; test it after the remaining lab setup is finished:

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

### Test a Tunnel before GitLab is installed

Use a temporary HTTP server on the MiniPC. This confirms the Tunnel, DNS, TLS,
and routing without requiring GitLab.

**Security warning:** `python3 -m http.server` exposes the files in its current
working directory, including directory listings. Never start it from `/home`,
the repository directory, or any directory containing SSH keys, credentials,
configuration files, or other private data. Use a dedicated directory
containing only a harmless test file:

```bash
mkdir -p /tmp/cloudflare-tunnel-test
printf 'Cloudflare Tunnel test OK\n' > /tmp/cloudflare-tunnel-test/index.html
cd /tmp/cloudflare-tunnel-test
python3 -m http.server 8080 --bind 127.0.0.1
```

In the Cloudflare dashboard, go to **Networking → Tunnels**, select your
tunnel, open the **Routes** tab, and choose **Add route → Published
application**. Configure:

```text
Hostname: tunnel-test.<YOUR_DOMAIN>
Service:  http://localhost:8080
```

From the MiniPC, verify the origin first:

```bash
curl http://127.0.0.1:8080
```

From a different network, such as mobile data, verify the public route:

```bash
dig tunnel-test.<YOUR_DOMAIN> CNAME +short
curl -i https://tunnel-test.<YOUR_DOMAIN>
```

The response should contain `Cloudflare Tunnel test OK`. Check the connector
if the public request fails:

```bash
cloudflared tunnel list
sudo journalctl -u cloudflared --no-pager -n 50
```

Stop the HTTP server with `Ctrl+C` and remove the temporary published
application route immediately after the test. If the home directory or another
private directory was accidentally exposed, stop the server, remove the route,
review access logs, and rotate any credentials or SSH keys that may have been
accessible.

If `cloudflared` runs in a container, `localhost` means that container; use a
reachable host/container service name instead.

Checklist:

- [x] Domain status **Active** in Cloudflare
- [x] Nameservers updated at registrar
- [ ] Subdomain A records or Tunnel routes defined
- [ ] API token created for cert-manager
- [ ] SSL mode documented

Create DNS records or Tunnel routes only when the corresponding service is
needed.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| DNS not resolving | Wait propagation; verify nameservers |
| SSL errors with orange cloud | Switch to DNS only until origin cert ready |
| 522/521 from Cloudflare | Origin unreachable — check port forward/Tunnel |
| 406 even when the origin is stopped | Check Cloudflare Rules, Workers, Access, and Tunnel routes for the hostname; an edge-generated response is being returned before GitLab is contacted |

#### Diagnose a fixed 406 response

If the response is identical while GitLab is running and while it is stopped,
the response is probably generated by Cloudflare rather than GitLab. Capture
the headers and redirect target:

```bash
curl -sS -D - -o /dev/null https://gitlab.<YOUR_DOMAIN>
curl -sS -D - -o /dev/null http://gitlab.<YOUR_DOMAIN>
dig gitlab.<YOUR_DOMAIN> A +short
dig gitlab.<YOUR_DOMAIN> CNAME +short
```

In the Cloudflare dashboard, inspect these areas for an exact
`gitlab.<YOUR_DOMAIN>` match:

1. **DNS → Records** — use either the server's current public IP or the
   intended `<TUNNEL_UUID>.cfargotunnel.com` target. Remove stale duplicate
   records.
2. **Zero Trust → Networks → Tunnels → Published applications** — confirm the
   route points to the actual local listener, for example
   `http://127.0.0.1:80`, and that no catch-all route precedes it.
3. **Rules → Redirect Rules** and **Workers & Pages → Overview** — disable
   hostname-specific redirects or Workers temporarily and retest.
4. **Zero Trust → Access → Applications** — confirm an Access policy is not
   handling the GitLab hostname unexpectedly.

An HTTP `Location` pointing to `https://255.255.255.255/` is invalid and must
be removed from the matching redirect rule, Worker, or Tunnel configuration.
After correcting the edge configuration, purge only the affected hostname's
cache if necessary and retest with:

```bash
curl -i https://gitlab.<YOUR_DOMAIN>/
```

Do not change GitLab's `external_url` or nginx settings until the hostname
reaches the origin; those settings cannot correct an edge-generated 406.

## Next

→ [Chapter 06 — Docker](./06-docker.md)
