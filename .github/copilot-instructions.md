# Copilot instructions

## Repository purpose and architecture

This repository is a documentation project for a small DevOps lab, not an application codebase. It documents a single Ubuntu 24.04 MiniPC that can later scale to multiple nodes.

The primary platform flow is:

1. Ubuntu host and network hardening
2. Docker for local/container build needs
3. GitLab CE, GitLab Runner, and the GitLab Container Registry
4. K3s as the Kubernetes cluster
5. ArgoCD watching a GitOps repository and synchronizing Kubernetes manifests
6. Traefik handling ingress and cert-manager issuing Let's Encrypt certificates through Cloudflare DNS-01
7. Drupal, WordPress, and Symfony workloads deployed in separate namespaces

The normal documentation path is chapters 00–25 in order. Chapter 26 is an independent, end-to-end Ubuntu → K3s → Traefik → ArgoCD → GitLab → Harbor installation path and does not depend on the earlier draft chapters.

## Build, test, and lint

No automated build, test, lint, or package-manager configuration exists in this repository. Changes are Markdown documentation changes.

For manual validation:

- Review the changed chapter's `Verify` section and run its commands on the lab MiniPC.
- Check Markdown links and chapter status manually when changing navigation or chapter content.
- There is no repository-supported single-test command; validate one chapter by following only that chapter's prerequisites, steps, and `Verify` section.

## Documentation structure and conventions

- Keep one chapter per file under `docs/` using `NN-short-kebab-title.md`.
- Preserve the chapter outline used throughout the repository: `Overview`, `Prerequisites`, `Goals`, `Steps`, `Verify`, optional `Troubleshooting`, and `Next`.
- Keep the two-digit chapter prefix aligned with the chapter number and update `docs/README.md` when adding or renumbering a chapter.
- Link prerequisites and the next chapter using relative Markdown links.
- Keep prose concise and operational. Prefer complete, copy-pasteable Linux commands over abstract descriptions.
- Use angle-bracket placeholders such as `<YOUR_DOMAIN>`, `<SERVER_IP>`, and `<GITLAB_TOKEN>` for environment-specific values. Do not introduce real IPs, credentials, API tokens, passwords, or private keys.
- Show `sudo` explicitly for privileged host commands; do not imply that routine work should be done as root.
- The default target is Ubuntu Server 24.04 LTS unless a chapter explicitly documents another version or compatibility case.
- Mark chapter maturity in `docs/README.md` with the repository’s existing statuses: `Draft`, `Review`, `Tested`, or `Production-ready`. Most chapters are currently `Draft`; do not imply hardware validation that has not occurred.
- Keep changes focused on the selected chapter. If a chapter changes a dependency, workflow, or status, update the directly affected index or cross-links as well.

## Infrastructure and GitOps conventions

- Treat GitLab as the source-control/CI platform and registry, and ArgoCD as the deployment controller. The intended delivery path is git push → GitLab CI test/build → registry image → GitOps manifest update → ArgoCD sync → Kubernetes rollout.
- Use K3s conventions such as `kubectl`, the default `local-path` storage class, and namespaces per application (`drupal`, `wordpress`, and `symfony`) unless the chapter states otherwise.
- Keep Traefik ingress configuration separate from cert-manager certificate configuration. Cloudflare is the DNS/edge provider and is also used for DNS-01 challenges.
- Prefer GitOps changes over direct cluster mutation for ongoing application configuration. Direct `kubectl apply` or `argocd app sync` commands are appropriate for bootstrap and verification examples.
- When documenting secrets, describe where they are stored (for example, a masked GitLab CI variable or Kubernetes secret) without embedding their values. Prefer Sealed Secrets, External Secrets, or SOPS for GitOps-managed secrets.
- Make destructive commands visibly explicit and include a recovery or backup reference when the surrounding documentation has one.
- Use the naming and command patterns already established in the chapters (`kubectl ... -n <namespace>`, `argocd app ...`, `sudo gitlab-ctl ...`, and `docker ...`).

## Editing workflow

Use `WORKING.md` as the detailed source of truth when these instructions need more context:

1. Select one chapter from `docs/README.md`.
2. Walk through its commands on the MiniPC and identify incorrect or missing steps.
3. Make a focused edit to the chapter.
4. Update the chapter status in `docs/README.md`.
5. Check prerequisite, previous/next, and related-chapter links.

