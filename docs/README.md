# DevOps Lab — GitLab + K3s + ArgoCD

Step-by-step guide to build a single-server development lab that grows into a cheap, production-like hosting platform for **Drupal**, **WordPress**, and **Symfony** sites.

## How to use this documentation

1. Work through chapters **in order** — later chapters assume earlier ones are done.
2. Replace placeholders like `<YOUR_DOMAIN>`, `<SERVER_IP>`, and `<SUBDOMAIN>` with your values.
3. Run every **Verify** section before moving on.
4. Review chapters one by one; content will be refined as the lab is built.

## Chapter index

| # | Chapter | Status |
|---|---------|--------|
| 00 | [Introduction](./00-introduction.md) | Draft |
| 01 | [Ubuntu 24 Installation](./01-ubuntu-24-installation.md) | Draft |
| 02 | [Initial Server Hardening](./02-initial-server-hardening.md) | Draft |
| 03 | [Network Configuration](./03-network-configuration.md) | Draft |
| 04 | [Digi Internet](./04-digi-internet.md) | Draft |
| 05 | [Cloudflare](./05-cloudflare.md) | Draft |
| 06 | [Docker](./06-docker.md) | Draft |
| 07 | [Git](./07-git.md) | Draft |
| 08 | [GitLab CE](./08-gitlab-ce.md) | Draft |
| 09 | [GitLab Runner](./09-gitlab-runner.md) | Draft |
| 10 | [Container Registry](./10-container-registry.md) | Draft |
| 11 | [Kubernetes (K3s)](./11-kubernetes-k3s.md) | Draft |
| 12 | [ArgoCD](./12-argocd.md) | Draft |
| 13 | [GitOps Repository](./13-gitops-repository.md) | Draft |
| 14 | [Traefik](./14-traefik.md) | Draft |
| 15 | [Cert Manager](./15-cert-manager.md) | Draft |
| 16 | [Drupal Platform](./16-drupal-platform.md) | Draft |
| 17 | [WordPress Platform](./17-wordpress-platform.md) | Draft |
| 18 | [Symfony Platform](./18-symfony-platform.md) | Draft |
| 19 | [CI/CD Pipeline](./19-cicd-pipeline.md) | Draft |
| 20 | [Monitoring](./20-monitoring.md) | Draft |
| 21 | [Backup](./21-backup.md) | Draft |
| 22 | [Security](./22-security.md) | Draft |
| 23 | [Scaling](./23-scaling.md) | Draft |
| 24 | [Troubleshooting](./24-troubleshooting.md) | Draft |
| 25 | [Appendix](./25-appendix.md) | Draft |

## Adaptations from the original plan

- **Added chapters 17–18** — WordPress and Symfony platforms (you listed all three CMS/frameworks as targets).
- **Renumbered 17→19 through 23→25** — CI/CD, monitoring, backup, security, scaling, troubleshooting, appendix.
- **Split concerns clearly** — Traefik (ingress) and cert-manager (certificates) stay separate; GitOps repo comes before app platforms.
- **Single-server first** — every chapter defaults to one MiniPC; scaling is deferred to chapter 23.

## Repository layout

```
ubuntu/
├── README.md                 # Project overview and quick links
├── WORKING.md                # How to work in this repository
├── .cursorrules              # AI/editor guidance
└── docs/
    ├── README.md             # This file — chapter index
    ├── 00-introduction.md
    ├── 01-ubuntu-24-installation.md
    └── ...
```
