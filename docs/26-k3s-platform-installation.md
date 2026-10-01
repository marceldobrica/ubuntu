# Standalone Platform Runbook — Ubuntu, K3s, Traefik, Argo CD, GitLab, and Harbor

## Purpose

This runbook installs a small, self-contained platform on an Ubuntu Server host:

1. Ubuntu Server
2. OpenVPN access layer
3. K3s Kubernetes
4. Traefik ingress
5. Argo CD
6. GitLab CE and GitLab Runner on the Ubuntu host
7. Harbor as the global OCI registry

Every stage has its own verification gate. Do not continue until the current
stage passes. Later stages are not required to verify an earlier stage.

This is a **single-node lab/reference installation**. GitLab and Harbor are
stateful and resource-intensive; use separate nodes, external databases, and
external object storage for production.

## Architecture

```text
Internet / LAN
      |
      | UDP 1194: OpenVPN
      | HTTPS: public application websites only
      v
Ubuntu Server
      |
      +-- OpenVPN (10.8.0.0/24)
      |     +-- Private administrator access
      +-- K3s
            +-- Traefik      (private services + selected public websites)
            +-- Argo CD      (VPN only)
            +-- GitLab CE    (host service, VPN only)
            +-- GitLab Runner (host service)
            +-- Harbor       (VPN only)
            +-- Websites     (public HTTPS where explicitly allowed)
```

Harbor is global from the infrastructure point of view: all projects use the
same endpoint. Harbor projects provide namespaces and permissions:

```text
registry.<DOMAIN>/platform/image:tag
registry.<DOMAIN>/applications/image:tag
registry.<DOMAIN>/libraries/image:tag
```

The security boundary is intentionally split:

- `argocd.<DOMAIN>`, `gitlab.<DOMAIN>`, and `registry.<DOMAIN>` are accessible
  only after connecting to OpenVPN.
- Websites deployed through Argo CD can use separate public hostnames such as
  `www.<DOMAIN>` or `app.<DOMAIN>`.
- Public website Ingress objects must be explicitly marked as public. Private
  services use an IP allow-list for the OpenVPN subnet.
- OpenVPN itself must be reachable from the internet, normally on UDP port
  `1194`; this is the only administrative entry point exposed publicly.

## Assumptions and resource planning

- Ubuntu Server 24.04 LTS, amd64
- One static IP address, referred to below as `<SERVER_IP>`
- A DNS zone you control, referred to below as `<DOMAIN>`
- At least 16 GB RAM and 4 vCPUs for a small lab
- At least 200 GB of fast storage
- Ports 1194/UDP and 443/TCP available; port 22 and Kubernetes API port 6443
  should be restricted to the VPN or a separate trusted administration network
- A non-root administrator named `labadmin`

GitLab runs as an Omnibus host service rather than inside K3s. This avoids a
circular dependency when rebuilding the cluster and leaves Harbor as the
cluster-wide OCI registry. The bundled GitLab PostgreSQL, Redis, and Gitaly
services below are intended only for a lab; use separate nodes or external
dependencies for production.

## 0. Prepare DNS and variables

Create DNS records that point to `<SERVER_IP>`:

```text
gitlab.<DOMAIN>     A    <SERVER_IP>
argocd.<DOMAIN>     A    <SERVER_IP>
registry.<DOMAIN>   A    <SERVER_IP>
```

On the server, set variables for the commands in this document:

```bash
export DOMAIN=example.com
export SERVER_IP=203.0.113.10
export GITLAB_HOST=gitlab.${DOMAIN}
export ARGOCD_HOST=argocd.${DOMAIN}
export REGISTRY_HOST=registry.${DOMAIN}
```

Do not use these example values in production.

### Security

- Use a private DNS zone or restrict access with firewall rules if this is an
  internal platform.
- Do not expose Kubernetes API port `6443` to the public internet. Allow it
  only from the VPN subnet or a separate trusted administrator network.
- Use DNS providers and accounts protected by MFA.

## 1. Install Ubuntu Server

Install Ubuntu Server 24.04 LTS using the official installer:

1. Boot the Ubuntu Server ISO in UEFI mode.
2. Use a static reservation or configure a static address after installation.
3. Set a unique hostname, for example `platform-01`.
4. Install **OpenSSH Server**.
5. Create the `labadmin` user.
6. Apply all updates and reboot.

From a workstation, connect to the server:

```bash
ssh labadmin@<SERVER_IP>
```

Install the base tools:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y ca-certificates curl git jq vim openssl \
  apt-transport-https software-properties-common
sudo reboot
```

### Verify Ubuntu before continuing

```bash
source /etc/os-release
test "$ID" = ubuntu
echo "$VERSION_ID"
systemctl is-active --quiet ssh
ip route
df -h /
free -h
```

Expected:

- Ubuntu `24.04`
- SSH is `active`
- A default route exists
- Enough free disk and memory for the planned workloads

If this gate fails, fix the operating system or network before installing
Kubernetes.

### Security

- Use the minimal Ubuntu Server installation and encrypt the disk where
  practical.
- Use a unique administrator account; do not permit routine administration as
  `root`.
- Prefer SSH keys over passwords and disable SSH password authentication after
  confirming key-based login.
- Enable unattended security updates or define a regular patching schedule.
- Restrict SSH with a firewall or VPN to trusted source networks.

## 2. Install and verify OpenVPN

OpenVPN is the private access layer for administration. Install and test it
before installing Kubernetes so that later services can be restricted to the
VPN independently.

Install OpenVPN and Easy-RSA:

```bash
sudo apt update
sudo apt install -y openvpn easy-rsa ufw
make-cadir "$HOME/easy-rsa"
cd "$HOME/easy-rsa"
```

Create the certificate authority, server certificate, and one client
certificate. Protect the CA and client private keys:

```bash
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
./easyrsa gen-req workstation nopass
./easyrsa sign-req client workstation
openvpn --genkey secret ta.key
```

Copy the server files:

```bash
sudo install -d -m 700 /etc/openvpn/server
sudo install -m 600 pki/private/server.key /etc/openvpn/server/
sudo install -m 644 pki/ca.crt pki/issued/server.crt pki/dh.pem \
  /etc/openvpn/server/
sudo install -m 600 ta.key /etc/openvpn/server/
```

Create `/etc/openvpn/server/server.conf`:

```conf
port 1194
proto udp
dev tun
user nobody
group nogroup
persist-key
persist-tun
topology subnet
server 10.8.0.0 255.255.255.0
push "route 10.8.0.0 255.255.255.0"
push "route <SERVER_PRIVATE_SUBNET> <NETMASK>"
keepalive 10 120
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-crypt ta.key
auth SHA256
cipher AES-256-GCM
data-ciphers AES-256-GCM:AES-128-GCM
duplicate-cn
explicit-exit-notify 1
verb 3
```

For a server behind a typical home or office router, the server might have an
address such as `192.168.100.25`. In that case:

```text
Server address:       192.168.100.25
Private LAN subnet:   192.168.100.0
Subnet mask:          255.255.255.0
CIDR notation:        192.168.100.0/24
```

The corresponding OpenVPN line is:

```conf
push "route 192.168.100.0 255.255.255.0"
```

Do not use the server's host address (`192.168.100.25`) as the subnet value;
use the network address (`192.168.100.0`). If your router uses a different
mask, use the actual values reported by the server:

```bash
ip -4 route
```

For example, this output:

```text
default via 192.168.100.1 dev eno1
192.168.100.0/24 dev eno1 proto kernel scope link src 192.168.100.25
```

means:

```conf
push "route 192.168.100.0 255.255.255.0"
```

If VPN clients need to reach other devices on the `192.168.100.0/24` LAN,
add a static route on the router:

```text
Destination: 10.8.0.0/24
Gateway:     192.168.100.25
```

This tells the router to send replies for OpenVPN clients back to the Ubuntu
server. Without this route, VPN clients can reach the Ubuntu server but
connections to other LAN devices may fail. An alternative is to configure
masquerading on the Ubuntu server, but a router route preserves the real VPN
client addresses and is preferred for a controlled network.

Also forward UDP port `1194` on the router to the server:

```text
WAN UDP 1194 → 192.168.100.25 UDP 1194
```

Do not use `duplicate-cn` when individual client identity and revocation are
required; remove it for production and issue one certificate per device.

Enable forwarding and start OpenVPN:

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/91-openvpn.conf
sudo sysctl --system
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server
```

### Create an OpenVPN client on an Ubuntu 20.04 workstation

The workstation does not need Ubuntu 24.04. Ubuntu 20.04 can use the OpenVPN
client packages supplied by its repositories. The workstation needs only the
CA certificate, the client certificate, the client private key, and the
`tls-crypt` key. **Never copy the CA private key** (`pki/private/ca.key`) to
the workstation.

The following commands in this subsection are run on the **workstation**,
unless the command is explicitly marked as a server command.

1. Install the client packages on Ubuntu 20.04:

   ```bash
   sudo apt update
   sudo apt install -y openvpn network-manager-openvpn network-manager-openvpn-gnome
   openvpn --version
   ```

   Ubuntu 20.04 commonly provides OpenVPN 2.4. OpenVPN 2.4 does not
   recognize the `data-ciphers` option used by OpenVPN 2.5 and newer; use
   the compatibility profile in step 5 when the version output starts with
   `2.4`.

2. Create a private directory for the client files:

   ```bash
   mkdir -p "$HOME/.config/openvpn"
   chmod 700 "$HOME/.config/openvpn"
   ```

3. Copy the files from the server to the workstation with `scp`. Run these
   commands on the workstation; replace `<SERVER_IP>` with the server's
   reachable address. If the workstation is outside the home network, use the
   router's public IP or a DNS name and ensure UDP port `1194` is forwarded to
   the server.

   ```bash
   scp labadmin@<SERVER_IP>:/home/labadmin/easy-rsa/pki/ca.crt \
     "$HOME/.config/openvpn/"
   scp labadmin@<SERVER_IP>:/home/labadmin/easy-rsa/pki/issued/workstation.crt \
     "$HOME/.config/openvpn/"
   scp labadmin@<SERVER_IP>:/home/labadmin/easy-rsa/pki/private/workstation.key \
     "$HOME/.config/openvpn/"
   scp labadmin@<SERVER_IP>:/home/labadmin/easy-rsa/ta.key \
     "$HOME/.config/openvpn/"
   ```

   If the Easy-RSA directory is not `/home/labadmin/easy-rsa`, run this on the
   server to find the files and use the actual paths:

   ```bash
   printf 'Home: %s\n' "$HOME"
   pwd
   ```

   The server administrator can also copy the files to a temporary location
   owned by `labadmin` before using `scp`:

   ```bash
   # Run on the server from the Easy-RSA directory.
   install -d -m 700 "$HOME/openvpn-workstation"
   install -m 644 pki/ca.crt pki/issued/workstation.crt \
     "$HOME/openvpn-workstation/"
   install -m 600 pki/private/workstation.key ta.key \
     "$HOME/openvpn-workstation/"
   ```

4. Set restrictive permissions on the workstation:

   ```bash
   chmod 600 "$HOME/.config/openvpn/"*
   ```

5. Create the client profile on the workstation:

   ```bash
   nano "$HOME/.config/openvpn/workstation.ovpn"
   ```

   Use this configuration. Replace `<VPN_SERVER_DNS_OR_PUBLIC_IP>` with a
   public DNS name or public IP address reachable by the workstation. Do not
   use `192.168.100.25` when connecting from outside the private LAN.

   ```conf
   client
   dev tun
   proto udp
   remote <VPN_SERVER_DNS_OR_PUBLIC_IP> 1194
   resolv-retry infinite
   nobind
   persist-key
   persist-tun
   remote-cert-tls server
   auth-nocache
   auth SHA256
   ncp-ciphers AES-256-GCM:AES-128-GCM
   cipher AES-256-GCM
   tls-crypt ta.key
   ca ca.crt
   cert workstation.crt
   key workstation.key
   verb 3
   ```

   The `ncp-ciphers` and `cipher` lines are required for an OpenVPN 2.4
   client. If `openvpn --version` reports 2.5 or newer, replace those two
   lines with:

   ```conf
   data-ciphers AES-256-GCM:AES-128-GCM
   data-ciphers-fallback AES-256-GCM
   ```

6. Test the profile from a terminal on the workstation:

   ```bash
   sudo openvpn --config "$HOME/.config/openvpn/workstation.ovpn"
   ```

   Keep this terminal open. A successful connection ends with a message such
   as `Initialization Sequence Completed`. In a second workstation terminal,
   verify the VPN interface and routes:

   ```bash
   ip addr show tun0
   ip route
   ping -c 3 10.8.0.1
   ssh labadmin@10.8.0.1
   ```

   Stop the foreground client with `Ctrl+C` after testing.

7. Optionally import the profile into NetworkManager so it can be started
   from the Ubuntu 20 desktop:

   ```bash
   nmcli connection import type openvpn \
     file "$HOME/.config/openvpn/workstation.ovpn"
   nmcli connection show
   nmcli connection up workstation
   ```

   If the imported connection has a generated name, use the name returned by
   `nmcli connection show`. To disconnect:

   ```bash
   nmcli connection down workstation
   ```

The OpenVPN profile and `workstation.key` are credentials. Store them with
mode `600`, do not email or commit them, and remove temporary copies from the
server after the transfer:

```bash
# Run on the server only after confirming the workstation connects.
rm -rf "$HOME/openvpn-workstation"
```

If the workstation is lost, revoke its certificate on the server and issue a
new certificate for the replacement device. Do not reuse the same client
certificate on multiple devices.

### Verify OpenVPN before continuing

From the workstation:

1. Disconnect from any existing VPN.
2. Connect using the client profile.
3. Confirm the client receives an address in `10.8.0.0/24`.
4. Confirm the server is reachable at its VPN address:

```bash
ping 10.8.0.1
ssh labadmin@10.8.0.1
```

On the server:

```bash
sudo systemctl is-active --quiet openvpn-server@server
ip addr show tun0
sudo journalctl -u openvpn-server@server --no-pager -n 50
```

Do not restrict SSH or other services to the VPN until this client connection
has been tested successfully from a separate workstation network.

Apply the host firewall only after the VPN connection has been tested. Keep
the current SSH session open while enabling it:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 1194/udp comment 'OpenVPN'
sudo ufw allow from 10.8.0.0/24 to any port 22 proto tcp comment 'SSH via VPN'
sudo ufw allow from 10.8.0.0/24 to any port 6443 proto tcp comment 'K3s API via VPN'
sudo ufw allow 80/tcp comment 'Public HTTP for redirects and ACME'
sudo ufw allow 443/tcp comment 'Public HTTPS websites and VPN services'
sudo ufw enable
sudo ufw status verbose
```

The firewall cannot distinguish hostnames on ports 80 and 443. Therefore,
Traefik's `vpn-only` middleware is also required to protect the private
hostnames. Public websites remain reachable on the same HTTPS ports, but their
Ingress objects must not reference that middleware.

### Security

- Expose only UDP `1194` publicly for OpenVPN; restrict SSH and port `6443` to
  `10.8.0.0/24` after VPN connectivity is confirmed.
- Protect the CA key offline. It should not remain on the VPN server after
  certificates have been issued.
- Issue one client certificate per person or device and revoke lost devices.
- Use MFA at the VPN layer if your OpenVPN deployment supports an external
  authentication provider.
- Keep OpenVPN and Ubuntu security updates current.
- Use a firewall with a default-deny policy, but test an active VPN session
  before closing administrative access.
- Keep a recovery path through the server console or hosting provider in case
  a firewall rule blocks VPN administration.

## 3. Install and verify K3s

Disable swap and load the kernel settings commonly needed by Kubernetes:

```bash
sudo swapoff -a
sudo sed -i.bak '/[[:space:]]swap[[:space:]]/ s/^/#/' /etc/fstab

cat <<'EOF' | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/90-kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Install a single-node K3s server. The bundled Traefik is disabled because the
next stage installs and verifies Traefik explicitly:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server --disable traefik --write-kubeconfig-mode 644" \
  sh -
```

Configure the current user's kubeconfig:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/rancher/k3s/k3s.yaml "$HOME/.kube/config"
sudo chown "$USER:$(id -gn)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"
export KUBECONFIG="$HOME/.kube/config"
```

Install Helm 3:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Verify K3s before continuing

```bash
sudo systemctl is-active --quiet k3s
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
helm version
```

The node must be `Ready`, `kubectl cluster-info` must succeed, and the K3s
system pods must settle without unexpected `CrashLoopBackOff` states.

### Security

- Protect `/etc/rancher/k3s/k3s.yaml`; it grants administrator access to the
  cluster. Keep the user copy at mode `600` and never commit it to Git.
- Do not publish the K3s kubeconfig or node token in tickets, chat, or CI logs.
- Keep K3s and its embedded components updated using a tested upgrade process.
- Use Kubernetes namespaces, RBAC, and NetworkPolicies as the platform grows.
- Restrict access to port `6443` with the host firewall.

## 4. Install and verify Traefik

Add the Traefik Helm repository:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create namespace traefik --dry-run=client -o yaml | kubectl apply -f -
```

Install Traefik with the Kubernetes Ingress and CRD providers enabled:

```bash
helm upgrade --install traefik traefik/traefik \
  --namespace traefik \
  --set providers.kubernetesCRD.enabled=true \
  --set providers.kubernetesIngress.enabled=true \
  --set service.type=LoadBalancer \
  --wait \
  --timeout 5m
```

Deploy a temporary test service. It does not depend on Argo CD, GitLab, Harbor,
DNS, or certificates:

```bash
kubectl create namespace ingress-test --dry-run=client -o yaml | kubectl apply -f -

kubectl -n ingress-test create deployment whoami \
  --image=traefik/whoami:v1.10.2 \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n ingress-test expose deployment whoami --port=80

cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  namespace: ingress-test
spec:
  ingressClassName: traefik
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
EOF
```

Find the service address:

```bash
kubectl get svc -n traefik
kubectl get ingress -n ingress-test
kubectl rollout status deployment/whoami -n ingress-test --timeout=180s
```

On a single-node K3s installation, ServiceLB normally exposes the
LoadBalancer service on the node. Test locally on the server:

```bash
curl -fsS -H 'Host: whoami.local' http://127.0.0.1/ | head
```

### Verify Traefik before continuing

```bash
helm status traefik -n traefik
kubectl rollout status deployment/traefik -n traefik --timeout=180s
kubectl get pods -n traefik
curl -fsS -H 'Host: whoami.local' http://127.0.0.1/ | grep -q 'Hostname'
```

If the local request fails, inspect `kubectl describe ingress -n ingress-test
whoami` and the Traefik logs. Do not proceed to application installation until
the test service is routed successfully.

### Security

- Expose only ports `80` and `443` through Traefik; do not expose dashboard or
  Kubernetes API endpoints publicly.
- Disable or protect the Traefik dashboard; never leave it unauthenticated.
- Use explicit host rules rather than catch-all routes for production services.
- Use HTTPS with trusted certificates and redirect HTTP to HTTPS.
- Keep Traefik and its Helm chart updated, and review generated routes after
  each application installation.
- Use an IP allow-list middleware on private hostnames so only `10.8.0.0/24`
  can reach Argo CD, GitLab, and Harbor.
- Do not apply that middleware to explicitly public website hostnames.

Example private-service middleware:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: vpn-only
  namespace: traefik
spec:
  ipAllowList:
    sourceRange:
      - 10.8.0.0/24
```

Reference `vpn-only@kubernetescrd` from the private service routes. Keep public
website routes separate and review every route before applying it.

## 5. Install and verify Argo CD

Install Argo CD from its upstream stable manifest. This stage can be verified
using port-forwarding and does not depend on Traefik or DNS:

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available \
  deployment/argocd-server -n argocd --timeout=10m
```

Retrieve the generated administrator password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
echo
```

In a separate terminal, create a temporary local access path:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

The `kubectl port-forward` command runs on the Ubuntu server and must remain
running. The server does not need a graphical environment or a web browser.
From your **workstation**, open a second terminal and create an SSH tunnel:

```bash
ssh -N -L 8080:127.0.0.1:8080 labadmin@<SERVER_IP>
```

Keep both terminals running. Then open this URL in a browser on your
workstation:

```text
https://127.0.0.1:8080
```

Accept the temporary certificate warning and log in with username `admin` and
the retrieved password. Change the password after the first login.

### Verify Argo CD before continuing

```bash
kubectl wait --for=condition=Ready pod --all -n argocd --timeout=10m
kubectl get svc -n argocd
```

The Argo CD pods must be ready and the local UI must load through the
port-forward. Public ingress is intentionally deferred so a Traefik or
certificate issue cannot hide an Argo CD installation problem.

### Security

- Change the initial `admin` password immediately and store it in a password
  manager.
- Prefer named Argo CD accounts or SSO for routine administration; limit use
  of the built-in admin account.
- Grant repositories and destination clusters only the permissions required by
  each team or application.
- Treat repository credentials, SSH keys, and Argo CD tokens as secrets.
- Do not expose the temporary port-forward or Argo CD API directly to the
  public internet.
- Publish the Argo CD hostname only through a Traefik route protected by the
  `vpn-only` middleware.

## 6. Install and verify GitLab CE and GitLab Runner

Install GitLab CE directly on the Ubuntu host. K3s already uses ports 80 and
443 through Traefik, so GitLab's bundled nginx listens on host port `8081`.
Traefik will route the private `gitlab.<DOMAIN>` hostname to that port after
the local installation check passes.

Install Docker for the runner's Docker executor:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker labadmin
```

Add the GitLab CE package repository and install the current CE package. Do
not enable GitLab's integrated Container Registry; Harbor is the global OCI
registry for this platform.

```bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://${GITLAB_HOST}" apt install -y gitlab-ce
```

Configure the host service:

```bash
sudo install -d -m 700 /etc/gitlab
sudo sed -i \
  -e "s|^external_url .*|external_url 'http://${GITLAB_HOST}'|" \
  -e "s|^# nginx\\['listen_port'\\].*|nginx['listen_port'] = 8081|" \
  -e "s|^# nginx\\['listen_https'\\].*|nginx['listen_https'] = false|" \
  /etc/gitlab/gitlab.rb
```

If either nginx setting was not already present in `/etc/gitlab/gitlab.rb`,
append these values explicitly:

```bash
sudo tee -a /etc/gitlab/gitlab.rb >/dev/null <<'EOF'
nginx['listen_port'] = 8081
nginx['listen_https'] = false
registry['enable'] = false
EOF
```

Apply the configuration and confirm that GitLab is listening only on the
internal HTTP port:

```bash
sudo gitlab-ctl reconfigure
sudo gitlab-ctl status
sudo ss -tlnp | grep ':8081'
```

Retrieve the initial root password and change it at the first login:

```bash
sudo cat /etc/gitlab/initial_root_password
```

Install and register GitLab Runner on the same Ubuntu host:

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
```

In GitLab, create a runner under **Admin Area → CI/CD → Runners** or at the
project/group level. Use a runner authentication token, then register it:

```bash
sudo gitlab-runner register
```

Use these values at the prompts:

| Prompt | Value |
|--------|-------|
| GitLab URL | `http://<SERVER_IP>:8081` during bootstrap, or the final HTTPS GitLab URL |
| Token | `<RUNNER_AUTHENTICATION_TOKEN>` |
| Description | `platform-docker` |
| Tags | `docker,platform` |
| Executor | `docker` |
| Default image | `docker:27` |

Enable the Docker socket in `/etc/gitlab-runner/config.toml`:

```toml
  [runners.docker]
    image = "docker:27"
    privileged = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
```

Restart and verify the runner:

```bash
sudo systemctl restart gitlab-runner
sudo gitlab-runner verify
sudo gitlab-runner list
```

### Route GitLab through Traefik

Create a Kubernetes Service and Endpoints object that points to the Ubuntu
host's internal address. This gives Traefik and Argo CD a stable Kubernetes
backend without installing GitLab in K3s:

```bash
kubectl create namespace platform --dry-run=client -o yaml | kubectl apply -f -

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: gitlab-host
  namespace: platform
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8081
---
apiVersion: v1
kind: Endpoints
metadata:
  name: gitlab-host
  namespace: platform
subsets:
  - addresses:
      - ip: ${SERVER_IP}
    ports:
      - name: http
        port: 8081
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab
  namespace: platform
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-vpn-only@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: ${GITLAB_HOST}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitlab-host
                port:
                  name: http
EOF
```

Configure trusted HTTPS for this hostname using the certificate strategy from
the Traefik and cert-manager chapters. Keep GitLab's host port `8081` private;
only Traefik should publish the final HTTPS endpoint.

The `vpn-only` middleware is for human access. Argo CD and the host-installed
GitLab Runner are not VPN clients, so allow the K3s node/pod CIDRs and the
server's internal address in the private route, or provide a separate
internal DNS/route for machine-to-machine access. Otherwise Argo CD will be
blocked while cloning the GitOps repository.

### Verify GitLab before continuing

```bash
sudo gitlab-ctl status
sudo gitlab-runner verify
curl -fsSI http://127.0.0.1:8081/users/sign_in | head
kubectl get ingress -n platform gitlab
```

GitLab must respond locally, the runner must be verified, and the Ingress must
reference `gitlab-host`. Do not debug Harbor or public ingress as a substitute
for a failed local GitLab check.

### Security

- Change the initial `root` password immediately and create named administrator
  accounts with MFA.
- Restrict sign-up, enforce MFA, and disable unused authentication methods.
- Use protected branches, protected tags, and protected CI/CD variables.
- Store runner, deploy-token, and registry credentials in masked, protected
  variables; never put them in repository files.
- Enable HTTPS before exposing GitLab beyond the trusted host or LAN.
- Plan GitLab backups and test restoration before storing important repositories.
- Protect the GitLab hostname with the `vpn-only` Traefik middleware.
- Use Harbor, not GitLab's integrated registry, for CI image pushes and
  Kubernetes image pulls.

## 7. Install and verify Harbor

Harbor is the global OCI registry. GitLab's integrated Container Registry
remains disabled. Add the official Harbor chart repository:

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
kubectl create namespace harbor --dry-run=client -o yaml | kubectl apply -f -
```

### Lab-only Harbor installation

The following configuration uses a ClusterIP service and Harbor's internal
HTTP endpoint for an installation check. It avoids depending on public DNS,
Traefik, or a certificate issuer. Do not expose this configuration directly to
an untrusted network.

```bash
helm upgrade --install harbor harbor/harbor \
  --namespace harbor \
  --timeout 15m \
  --set expose.type=clusterIP \
  --set expose.tls.enabled=false \
  --set externalURL="http://${REGISTRY_HOST}" \
  --set persistence.enabled=true
```

Check the release and forward Harbor locally:

```bash
helm status harbor -n harbor
kubectl get pods -n harbor
kubectl -n harbor port-forward svc/harbor-portal 8082:80
```

The `kubectl port-forward` command runs on the Ubuntu server. From your
**workstation**, open a second terminal and create the SSH tunnel:

```bash
ssh -N -L 8082:127.0.0.1:8082 labadmin@<SERVER_IP>
```

Keep both terminals running. Then open `http://127.0.0.1:8082` in a browser on
your workstation. The default Harbor administrator is `admin`;
retrieve the generated password from the chart secret:

```bash
kubectl -n harbor get secret harbor-core \
  -o jsonpath='{.data.HARBOR_ADMIN_PASSWORD}' | base64 -d
echo
```

The exact secret key can vary with the chart version. If that key is absent,
inspect the secret keys without printing values:

```bash
kubectl -n harbor get secret harbor-core \
  -o jsonpath='{.data}' | jq -r 'keys[]'
```

### Configure Harbor for real registry use

Before pushing images from Docker or containerd, configure all of the
following:

- A trusted certificate for `registry.<DOMAIN>`
- A Traefik Ingress for Harbor
- `externalURL` set to `https://registry.<DOMAIN>`
- Harbor TLS enabled, or a TLS secret supplied to the chart
- DNS for `registry.<DOMAIN>` pointing at the server
- Persistent storage and backups
- Harbor projects and robot accounts

Do not solve certificate problems by permanently configuring Docker as an
insecure registry.

Create a Harbor project named `applications` in the UI, then the intended
image path becomes:

```text
registry.<DOMAIN>/applications/my-image:latest
```

### Verify Harbor before continuing

```bash
helm status harbor -n harbor
kubectl get pods -n harbor
kubectl get pvc -n harbor
curl -fsSI http://127.0.0.1:8082/ | head
```

The Harbor pods must be ready, persistent volume claims must be bound, and the
portal must respond through the local port-forward. This verifies Harbor
independently from Traefik, DNS, TLS, GitLab, and Argo CD.

### Security

- Change the initial Harbor `admin` password and enable MFA or external
  identity integration where available.
- Use Harbor robot accounts for CI instead of personal passwords or long-lived
  administrator credentials.
- Create separate Harbor projects and least-privilege roles for teams and
  environments.
- Enable vulnerability scanning, immutable tags, retention policies, and image
  signing where those controls are required.
- Use trusted TLS for all registry traffic. Do not permanently configure
  Docker or containerd to trust an insecure registry.
- Back up Harbor metadata and persistent storage, and test recovery.
- Protect the Harbor hostname with the `vpn-only` Traefik middleware. CI jobs
  running outside the VPN need a private runner path or a deliberately
  controlled alternative; do not make Harbor public just to simplify CI.

## 8. Final integration checks

Only after every previous gate passes should you connect the services through
Traefik and trusted HTTPS:

```bash
kubectl get ingress -A
kubectl get ingressclass
kubectl get pods -A
```

Recommended final hostnames:

```text
VPN only:
  https://gitlab.<DOMAIN>
  https://argocd.<DOMAIN>
  https://registry.<DOMAIN>

Public only when explicitly intended:
  https://www.<DOMAIN>
  https://app.<DOMAIN>
```

For private hostnames, connect to OpenVPN first and verify that the route uses
the VPN address. From a device not connected to OpenVPN, requests to the
private hostnames must fail. For public websites, verify access both with and
without OpenVPN.

For Harbor, always test an actual registry operation after TLS is configured:

```bash
docker login registry.<DOMAIN>
docker pull alpine:3.20
docker tag alpine:3.20 registry.<DOMAIN>/applications/alpine:test
docker push registry.<DOMAIN>/applications/alpine:test
docker pull registry.<DOMAIN>/applications/alpine:test
```

### Security

- Verify every public hostname resolves to the intended server and certificate.
- Permit inbound traffic only for OpenVPN UDP `1194`, public HTTPS website
  routes, and any explicitly approved monitoring or DNS services.
- Permit SSH and Kubernetes API access only from the VPN subnet or a separate
  trusted administrator network.
- Review Kubernetes Ingress objects for accidental public exposure.
- Require an explicit review before adding a public hostname to an Argo CD
  application. Public websites must not share private admin hostnames.
- Remove temporary test deployments, port-forwards, and unused namespaces.
- Monitor authentication logs, image pulls, failed logins, certificate expiry,
  and storage usage.

## Failure isolation

Use the first failing gate as the boundary:

| Failing stage | Inspect first |
|---|---|
| Ubuntu | `systemctl`, network route, disk, RAM |
| K3s | `systemctl status k3s`, `kubectl get nodes`, K3s logs |
| Traefik | Traefik pods, LoadBalancer service, temporary `whoami` ingress |
| Argo CD | Argo CD pods and port-forward |
| GitLab | `gitlab-ctl status`, local port 8081, and the `platform` Ingress |
| Harbor | Helm status, Harbor namespace pods, PVCs, portal port-forward |
| OpenVPN | certificate/profile, UDP 1194, routes, forwarding, firewall |
| Final HTTPS | DNS, certificate chain, Traefik routes, VPN middleware, firewall |

Do not troubleshoot a later stage until the earlier stage's local verification
passes.

## Sources

- [Ubuntu Server](https://ubuntu.com/download/server)
- [K3s Quick Start](https://docs.k3s.io/quick-start)
- [Traefik Helm chart](https://artifacthub.io/packages/helm/traefik/traefik)
- [Argo CD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [GitLab CE installation](https://about.gitlab.com/install/)
- [GitLab Runner installation](https://docs.gitlab.com/runner/install/)
- [Harbor Helm chart](https://github.com/goharbor/harbor-helm)
