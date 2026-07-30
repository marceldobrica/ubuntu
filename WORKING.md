# Working in this repository

Instructions for maintaining and extending the DevOps lab documentation.

## Purpose

This repo documents a **development lab** that mirrors a future production setup:

- **Single MiniPC** to start (control plane + worker + GitLab + Runner)
- **K3s** for Kubernetes
- **GitLab CE + Runner** for CI
- **ArgoCD** for GitOps CD
- **Traefik + cert-manager** for HTTPS ingress with Let's Encrypt via Cloudflare DNS
- **Drupal, WordPress, Symfony** as reference application platforms

The lab is reachable from the internet through **Cloudflare** (domain purchased there, subdomains per service/site).

## Conventions

### File naming

- One chapter per file: `docs/NN-short-kebab-title.md`
- Two-digit prefix matches chapter number (`00`–`25`)
- Lowercase, hyphen-separated filenames

### Chapter structure

Every chapter should follow this outline:

```markdown
# Chapter NN — Title

## Overview
Brief description of what this chapter accomplishes.

## Prerequisites
Links to prior chapters that must be completed.

## Goals
Checklist of outcomes when this chapter is done.

## Steps
### 1. Step name
Commands and explanations.

## Verify
Commands to confirm success.

## Troubleshooting
Common issues (optional but encouraged).

## Next
Link to the following chapter.
```

### Writing style

- **Copy-pasteable commands** — full commands, no `...` omissions
- **Placeholders** — use angle brackets: `<YOUR_DOMAIN>`, `<SERVER_IP>`, `<GITLAB_TOKEN>`
- **Concise prose** — explain *why* only when non-obvious
- **Ubuntu 24.04 LTS** — default target unless noted
- **Root vs sudo** — show `sudo` explicitly; avoid running daily tasks as root

### Status markers

Use these in chapter headers or the index table when reviewing:

| Marker | Meaning |
|--------|---------|
| Draft | Written but not tested on hardware |
| Review | Ready for your walkthrough |
| Tested | Verified on the lab MiniPC |
| Production-ready | Safe to follow for prod-like setup |

## Editing workflow

1. **Pick one chapter** from [docs/README.md](./docs/README.md).
2. **Walk through it** on the MiniPC; note gaps, wrong commands, or missing steps.
3. **Update the md file** with fixes; keep diffs focused on that chapter.
4. **Update the status** in `docs/README.md`.
5. **Cross-link** — if a chapter references a subdomain or secret defined elsewhere, link to that chapter.

## Placeholder registry

Keep a local (uncommitted) file or password manager entry for secrets. **Never commit**:

- Cloudflare API tokens
- GitLab root password / tokens
- SSH private keys
- Database passwords

Document *where* secrets live (e.g. "Cloudflare API token in `cert-manager` K8s secret") without embedding values.

## Adding a new site (Drupal / WordPress / Symfony)

After chapters 00–19 are in place:

1. Add application manifests to the **GitOps repo** (chapter 13).
2. Add CI job in the **application repo** (chapter 19).
3. Create DNS record in **Cloudflare** (chapter 05).
4. ArgoCD syncs; **Traefik + cert-manager** issue the certificate (chapters 14–15).

## Adding a worker node

Follow chapter 23 only after the single-node cluster is stable.

## AI-assisted editing (Cursor)

The `.cursorrules` file guides AI edits:

- Preserve Markdown structure
- Prefer practical Linux/Ubuntu commands
- Do not expand scope into unrelated tooling

When asking AI to update a chapter, specify the chapter number and what you tested.

## Git commits

- One chapter per commit when possible: `docs: refine chapter 08 GitLab CE install`
- Do not commit secrets or environment-specific IPs unless intentionally documenting a public example

## Review checklist (per chapter)

- [ ] Prerequisites listed and correct
- [ ] All commands tested or marked Draft
- [ ] Verify section exists and is runnable
- [ ] Placeholders documented
- [ ] Links to next/previous chapter work
- [ ] Status updated in `docs/README.md`
