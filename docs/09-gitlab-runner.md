# Chapter 09 — GitLab Runner

## Overview

Install GitLab Runner on the MiniPC with the Docker executor so CI jobs build and push container images.

## Prerequisites

- [Chapter 08 — GitLab CE](./08-gitlab-ce.md)
- [Chapter 06 — Docker](./06-docker.md)

## Goals

- [ ] GitLab Runner installed and registered
- [ ] Docker executor configured
- [ ] Runner picks up jobs
- [ ] Cache configured (optional)

## Steps

### 1. Install GitLab Runner

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner
```

### 2. Add gitlab-runner user to docker group

```bash
sudo usermod -aG docker gitlab-runner
```

### 3. Get registration token

In GitLab UI:

**Admin → CI/CD → Runners → New instance runner**

Or for a group/project: **Settings → CI/CD → Runners**

Copy the registration token (legacy) or use runner authentication token (GitLab 16+).

### 4. Register runner

```bash
sudo gitlab-runner register
```

Prompts:

| Prompt | Value |
|--------|-------|
| GitLab URL | `https://gitlab.<YOUR_DOMAIN>` |
| Token | `<RUNNER_TOKEN>` |
| Description | `minipc-docker` |
| Tags | `docker,minipc` |
| Executor | `docker` |
| Default image | `docker:24` |

### 5. Configure Docker executor

Edit `/etc/gitlab-runner/config.toml`:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

Example `[runners.docker]` section:

```toml
  [runners.docker]
    tls_verify = false
    image = "docker:24"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
```

`privileged = true` is required for Docker-in-Docker builds. Accept the security trade-off in a lab context.

### 6. Cache (optional)

Use local cache volume or S3-compatible storage. Minimal local setup:

```toml
  [runners.cache]
    Type = "filesystem"
    Path = "/cache"
    Shared = true
```

```bash
sudo mkdir -p /cache
sudo chown gitlab-runner:gitlab-runner /cache
```

### 7. Restart runner

```bash
sudo gitlab-runner restart
sudo gitlab-runner verify
```

## Verify

Create `.gitlab-ci.yml` in a test project:

```yaml
test-runner:
  tags:
    - minipc
  image: alpine
  script:
    - echo "Runner OK"
    - uname -a
```

Pipeline should succeed; runner shows green in GitLab UI.

```bash
sudo gitlab-runner list
sudo gitlab-runner status
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Jobs stuck pending | Check tags match; runner not paused |
| Docker socket errors | Verify `gitlab-runner` in `docker` group |
| dind failures | Ensure `privileged = true` |

## Next

→ [Chapter 10 — Container Registry](./10-container-registry.md)
