# Chapter 06 — Docker

## Overview

Install Docker Engine, Docker Compose plugin, and BuildKit for local builds and GitLab Runner's Docker executor.

## Prerequisites

- [Chapter 05 — Cloudflare](./05-cloudflare.md)

## Goals

- [x] Docker Engine installed
- [x] Docker Compose v2 plugin available
- [x] BuildKit enabled
- [x] User in `docker` group

## Steps

### 1. Remove old packages (if any)

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true
```

### 2. Add Docker official repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

### 3. Install Docker

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### 4. Enable and start Docker

```bash
sudo systemctl enable --now docker
```

### 5. Add user to docker group

```bash
sudo usermod -aG docker labadmin
newgrp docker
```

Log out and back in if group membership does not apply.

### 6. Enable BuildKit

```bash
mkdir -p ~/.docker
cat >> ~/.docker/config.json << 'EOF'
{
  "features": {
    "buildkit": true
  }
}
EOF
```

Or system-wide in `/etc/docker/daemon.json`:

```bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "features": {
    "buildkit": true
  },
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
sudo systemctl restart docker
```

### 7. Registry login (GitLab — after chapter 08)

Placeholder for later:

```bash
docker login registry.<YOUR_DOMAIN>
# Or gitlab.<YOUR_DOMAIN>:5050 depending on GitLab config
```

## Verify

```bash
docker --version
docker compose version
docker run --rm hello-world
DOCKER_BUILDKIT=1 docker build -t test-buildkit - << 'EOF'
FROM alpine
RUN echo "BuildKit OK"
EOF
docker rmi test-buildkit
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Permission denied on docker.sock | Re-login after `usermod -aG docker` |
| BuildKit not used | Export `DOCKER_BUILDKIT=1` in shell profile |

## Next

→ [Chapter 07 — Git](./07-git.md)
