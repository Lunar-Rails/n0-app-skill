# n0-app-skill

A Claude Code skill for analyzing codebases and generating everything needed to deploy them on [N0](https://nzero.pro) — a self-hosted team platform with per-workspace app hosting.

> **This repository is the single source of truth for the n0-app skill.** Copies shipped inside N0 agent sandboxes (`/workspace/skills/n0-app/SKILL.md`) and local installs are mirrors of this repo — contribute changes here.

## What it does

When triggered, this skill:

1. **Analyzes** a repository to detect the tech stack, framework, ports, databases, and dependencies
2. **Decides the image strategy**: upstream pre-built images, a custom `Dockerfile`, OR a zero-build `config_files` app (no CI needed)
3. **Generates** an `n0-app.json` manifest describing all services
4. **Generates** a `.gitea/workflows/build-and-push.yml` CI workflow for building and pushing Docker images
5. **Validates** the output against N0's manifest schema
6. **Deploys** via the N0 API (import definition → deploy instance)

## Installation

Add this skill to your Claude Code project by adding the following to your `.claude/settings.json`:

```json
{
  "skills": [
    {
      "source": "github:Lunar-Rails/n0-app-skill",
      "name": "n0-app"
    }
  ]
}
```

Or copy `SKILL.md` into your project's `.claude/skills/n0-app/` directory.

## Trigger phrases

- "make this deployable"
- "create app manifest"
- "n0 app"
- "deploy this on n0"
- "containerize this"
- "generate Dockerfile"

## What gets generated

| File | Purpose |
|------|---------|
| `n0-app.json` | App manifest (services, ports, env vars, volumes) |
| `Dockerfile` | Multi-stage Docker build (if no upstream image exists) |
| `.dockerignore` | Excludes unnecessary files from the build |
| `.gitea/workflows/build-and-push.yml` | CI pipeline: build → push → import into k3s |
| `migration.sql` | Database migration (if the app needs a DB table) |

## N0 Platform Overview

N0 deploys apps as Docker containers on Kubernetes (k3s), with:

- **Per-workspace isolation** — each workspace gets its own Gitea, Supabase, and app namespace
- **Automatic SSO** — Caddy reverse proxy injects `X-Webauth-*` headers
- **Gitea Actions CI/CD** — build and push images to workspace container registry
- **Vault secrets management** — secrets stored in HashiCorp Vault, injected at deploy time
- **Caddy routing** — automatic TLS, DNS, and reverse proxy setup

## License

MIT
