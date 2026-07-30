# DevOps Lab — Ubuntu MiniPC

Documentation for building a **GitLab + GitLab Runner + K3s + ArgoCD** development lab on Ubuntu 24.04, exposed via Cloudflare, hosting **Drupal**, **WordPress**, and **Symfony** sites.

## Start here

1. Read [WORKING.md](./WORKING.md) — how to use and extend this repository
2. Follow chapters in order from [docs/README.md](./docs/README.md)

## Quick links

| Chapter | Topic |
|---------|-------|
| [00 — Introduction](./docs/00-introduction.md) | Goals, architecture, hardware |
| [01 — Ubuntu install](./docs/01-ubuntu-24-installation.md) | Fresh OS setup |
| [08 — GitLab CE](./docs/08-gitlab-ce.md) | Git + CI platform |
| [11 — K3s](./docs/11-kubernetes-k3s.md) | Kubernetes cluster |
| [12 — ArgoCD](./docs/12-argocd.md) | GitOps CD |
| [19 — CI/CD](./docs/19-cicd-pipeline.md) | Full pipeline |
| [24 — Troubleshooting](./docs/24-troubleshooting.md) | When things break |

## Adaptations from original outline

- Added **WordPress** (ch. 17) and **Symfony** (ch. 18) platform chapters
- Renumbered CI/CD through Appendix (19–25)
- All chapters are **Draft** until tested on your MiniPC

## Legacy notes

Earlier Ubuntu install notes (USB creator, UEFI `nolapic`, package migration) are incorporated into [Chapter 01](./docs/01-ubuntu-24-installation.md).
