---
name: n0-app
description: >-
  Analyze a repository and generate a n0-app.json manifest and
  Dockerfile to make it deployable on N0's hosted apps platform.
  Trigger on: "make this deployable", "create app manifest", "n0 app",
  "deploy this on n0", "containerize this", "generate Dockerfile".
effort: high
user-invocable: true
argument-hint: "[optional: path to repo or description of the app]"
---

# N0 App Skill

Analyze a codebase and generate everything needed to deploy it as a hosted app on N0.

## What This Skill Does

1. **Analyzes** the repository to detect the tech stack, framework, ports, databases, and dependencies
2. **Determines the image strategy**: use upstream pre-built images OR generate a custom `Dockerfile`
3. **Generates** a `n0-app.json` manifest describing all services
4. **Generates** a `.gitea/workflows/build-and-push.yml` Gitea Actions workflow (build from source OR mirror upstream images)
5. **Validates** the output against N0's manifest schema

## CRITICAL: Image References

**N0 does NOT build Docker images. It only pulls pre-built images from registries.**

This means:
- Custom app images **MUST** be pre-built and pushed to a container registry
- The `image` field in `n0-app.json` **MUST** use the full registry path
- The registry is **this workspace's own Gitea container registry**. Its host is provided
  as the org-level Gitea Actions **variable `REGISTRY`** (auto-provisioned for every
  workspace — e.g. `127.0.0.1:30083`). Never hardcode a registry hostname.
- In **workflow YAML**, reference it as `${{ vars.REGISTRY }}`.
- In the **`n0-app.json` image field** (which is NOT a workflow and has no `${{ }}`
  substitution), write the **literal value** of the `REGISTRY` variable. Look it up with
  `GET /api/v1/orgs/{org}/actions/variables/REGISTRY` and paste the value in.
- Use Gitea Actions to automatically build and push the image on every commit.

**Correct image format** (`<REGISTRY>` = the literal value of the `REGISTRY` org variable):
```
<REGISTRY>/{org}/{repo}:{tag}
```

**Example** (if `REGISTRY` resolves to `127.0.0.1:30083`):
```json
"image": "127.0.0.1:30083/clovrlabs/my-app:latest"
```

**NEVER use bare image names** like `"my-app:latest"` — the platform will try to pull from Docker Hub and fail.

Public images from Docker Hub (e.g., `postgres:16-alpine`, `redis:7-alpine`, `nginx:alpine`) are fine as-is.

Upstream pre-built images from other registries (e.g., `ghcr.io/org/app:stable`) are also fine — see "Pre-built Upstream Images" below.

## Workflow

Always follow this order:

### Step 1: Analyze the Repository

Scan the codebase to determine:

- **Language & framework**: Check `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `Gemfile`, `pom.xml`, `composer.json`, etc.
- **Build system**: Vite, Webpack, Next.js, Create React App, etc.
- **Entry point**: `npm start`, `python manage.py runserver`, `go run main.go`, etc.
- **Port**: Check for `PORT` env var usage, hardcoded ports, or framework defaults
- **Database needs**: Check for database connection strings, ORM configs, migration files
- **External services**: Redis, S3, email, message queues
- **Existing Docker setup**: `Dockerfile`, `docker-compose.yml`, `.dockerignore`
- **Existing self-host setup**: Check for `.docker/selfhost/`, `self-host/`, `contrib/`, or similar directories with compose files and `.env.example` — these are the authoritative reference for deployment config
- **Environment variables**: `.env.example`, `.env.sample`, or env var references in code
- **Pre-built images**: Check if the project publishes container images to GHCR, Docker Hub, or another registry (look at GitHub Packages, README badges, CI workflows)
- **API routes**: Check for API endpoints that could be exposed as connectors (Express routes, Django URLs, FastAPI endpoints, Flask routes, Go handlers)
- **Default branch**: Check which branch is the default (`main`, `master`, `canary`, `develop`, etc.)

```bash
# Key files to check
ls -la
cat package.json 2>/dev/null || cat requirements.txt 2>/dev/null || cat go.mod 2>/dev/null
cat .env.example 2>/dev/null || cat .env.sample 2>/dev/null
cat Dockerfile 2>/dev/null
cat docker-compose.yml 2>/dev/null

# Check for self-host configs (IMPORTANT — use these as ground truth)
ls .docker/selfhost/ 2>/dev/null || ls self-host/ 2>/dev/null || ls contrib/docker/ 2>/dev/null
cat .docker/selfhost/compose.yml 2>/dev/null || cat self-host/docker-compose.yml 2>/dev/null

# Check for pre-built images in CI
cat .github/workflows/*.yml 2>/dev/null | grep -E 'ghcr\.io|docker\.io|push.*image'

# Check the default branch
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
```

### Step 2: Decide the Image Strategy

**Choose ONE of these two paths:**

#### Path A: Use Upstream Pre-built Images (preferred when available)

Use this path when:
- The project publishes official container images (GHCR, Docker Hub, etc.)
- The build process is complex (monorepo, native modules, Rust/C++ compilation)
- The project has an official self-host compose file

**When using upstream images, you MUST inspect the container before writing the manifest:**

```bash
# Check what files exist at runtime
docker run --rm --entrypoint="" <image>:<tag> ls -la /app/ 2>/dev/null

# Check the default CMD, entrypoint, and env vars
docker inspect <image>:<tag> --format='CMD: {{json .Config.Cmd}}'
docker inspect <image>:<tag> --format='ENTRYPOINT: {{json .Config.Entrypoint}}'
docker inspect <image>:<tag> --format='ENV: {{json .Config.Env}}'

# Check if there's an entrypoint script
docker run --rm --entrypoint="" <image>:<tag> cat /usr/local/bin/docker-entrypoint.sh 2>/dev/null
```

**NEVER guess environment variables or file paths.** Always verify against:
1. The upstream self-host compose file (ground truth)
2. The actual container filesystem (`docker run --rm --entrypoint="" ... ls/cat`)
3. The project's self-hosting documentation

Skip to Step 3 (manifest generation) — no Dockerfile needed.

#### Path B: Build a Custom Dockerfile

Use this path when:
- No pre-built images are available
- You're deploying a custom/internal app
- The app is a simple SPA, API server, or static site

Continue to Step 2b below.

### Step 2b: Generate the Dockerfile

Create a multi-stage Dockerfile optimized for the detected stack.

**General principles:**
- Use specific image tags (not `latest`) for reproducibility
- Multi-stage builds to minimize final image size
- Copy dependency files first for layer caching
- Run as non-root user when possible
- Use `.dockerignore` to exclude unnecessary files

**Common patterns:**

#### Node.js (React/Vue/Next.js SPA)
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL=""
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# SPA fallback + optional API proxy
RUN printf 'server {\n\
    listen 80;\n\
    root /usr/share/nginx/html;\n\
    index index.html;\n\
    location / {\n\
        try_files $uri $uri/ /index.html;\n\
    }\n\
}\n' > /etc/nginx/conf.d/default.conf
EXPOSE 80
```

#### Node.js (Express/Fastify/API server)
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production
COPY . .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

#### Python (Django/FastAPI/Flask)
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN python manage.py collectstatic --noinput 2>/dev/null || true
EXPOSE 8000
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

#### Go
```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /server .

FROM alpine:3.19
COPY --from=build /server /server
EXPOSE 8080
CMD ["/server"]
```

#### Static site (HTML/CSS/JS)
```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```

### Step 3: Generate n0-app.json

Create the manifest based on analysis. The manifest MUST be valid JSON.

**If the project has an upstream self-host compose file, translate it directly.**
Map compose services, environment variables, volumes, depends_on, and commands
into the n0-app.json format. Do NOT invent env vars or add options
that aren't in the upstream compose — this is the #1 cause of deployment failures.

#### Manifest Schema

```json
{
  "name": "Human-readable App Name",
  "slug": "app-name-lowercase",
  "description": "One-line description of the app",
  "badge": "Self-hosted",
  "icon": {
    "letter": "A",
    "bg": "bg-indigo-500/20",
    "color": "text-indigo-400"
  },
  "open_btn_class": "bg-indigo-500/20 hover:bg-indigo-500/30 text-indigo-400",
  "entrypoint": "web",
  "cpu": 1.0,
  "services": {
    "service-name": {
      "image": "registry/image:tag",
      "port": 3000,
      "env": {},
      "volumes": {},
      "config_files": {},
      "depends_on": [],
      "memory_mb": 256,
      "cpu": 0.5,
      "command": []
    }
  },
  "connectors": [
    {
      "slug": "my-api",
      "name": "My App API",
      "description": "Access my app's data",
      "auth_type": "platform_identity",
      "icon_emoji": "📦",
      "base_path": "/api/v1",
      "scopes": ["read", "write"]
    }
  ]
}
```

#### Field Reference

**Top-level fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | string | Display name in the app catalog |
| `slug` | Yes | string | Unique identifier, lowercase, hyphens only. Used as default subdomain |
| `description` | Yes | string | Short description (1-2 sentences) |
| `badge` | No | string | Badge text: `"Self-hosted"`, `"Internal"`, or `"Community"` (default) |
| `icon` | No | object | `{ letter, bg, color }` — Tailwind classes for catalog icon |
| `open_btn_class` | No | string | Tailwind classes for the "Open" button |
| `entrypoint` | No | string | Which service to expose to the internet. Default: `"web"` |
| `cpu` | No | number | Default CPU limit per service. Default: `1.0` |
| `connectors` | No | array | Connectors this app publishes (see "App-Published Connectors" below) |

**Service fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `image` | Yes | string | Full Docker image path with tag. For custom images use `<REGISTRY>/{org}/{repo}:{tag}` where `<REGISTRY>` is the literal value of the org-level `REGISTRY` Actions variable (e.g. `127.0.0.1:30083`). For public images use `registry/image:tag` (e.g., `postgres:16-alpine`). For upstream pre-built images use the full registry path (e.g., `ghcr.io/org/app:stable`). |
| `port` | Yes | number | Container port the service listens on |
| `env` | No | object | Environment variables as key-value strings |
| `volumes` | No | object | Named volume -> container path mapping |
| `config_files` | No | object | Files to write into volumes before container starts |
| `depends_on` | No | string[] | Service names that must start before this one |
| `memory_mb` | No | number | Memory limit in MB. Default: `256` |
| `cpu` | No | number | CPU limit (fractional). Default: `0.5` |
| `command` | No | string[] | Override the default container command |

**Template variables** (replaced at deploy time in `env` values):

| Variable | Replaced With |
|----------|--------------|
| `{{APP_URL}}` | `https://{subdomain}.apps.privateprompt.tech` |
| `{{APP_DOMAIN}}` | `{subdomain}.apps.privateprompt.tech` |
| `{{APP:<slug>:<field>}}` | Cross-app reference (see below) |

#### Cross-App Environment References

Apps can reference other running apps in their environment variables using the `{{APP:<slug>:<field>}}` syntax. This is resolved at deploy time.

| Pattern | Resolves To | Example |
|---------|-------------|---------|
| `{{APP:gitea:URL}}` | Public URL of the app | `https://gitea.apps.privateprompt.tech` |
| `{{APP:gitea:INTERNAL}}` | Docker-internal hostname | `gitea-gitea` (container name) |
| `{{APP:gitea:PORT}}` | Container port as string | `3000` |

**Example usage in a manifest:**
```json
{
  "env": {
    "GIT_SERVER_URL": "{{APP:gitea:URL}}",
    "GIT_INTERNAL_HOST": "{{APP:gitea:INTERNAL}}",
    "GIT_PORT": "{{APP:gitea:PORT}}"
  }
}
```

If the referenced app is not running or doesn't exist, the deploy validation will flag it as a warning.

#### Access Control in Manifests

You can set the default access level for an app in the manifest:

```json
{
  "name": "My Public App",
  "slug": "my-app",
  "access_level": "public"
}
```

| Value | Who can access |
|-------|---------------|
| `"workspace"` | Workspace members only (default) |
| `"restricted"` | Specific allowed users only |
| `"public"` | Anyone, no auth required |

The access level is auto-synced from the manifest on each deploy — no need to manually update the DB.

#### Config Files

Use `config_files` to write configuration files into named volumes before the container starts.
The key must match a volume name defined in `volumes`.

```json
{
  "volumes": {
    "nginx-conf": "/etc/nginx/conf.d"
  },
  "config_files": {
    "nginx-conf": {
      "default.conf": "server {\n    listen 80;\n    root /usr/share/nginx/html;\n}\n"
    }
  }
}
```

### Step 4: Validate

After generating the files, validate:

1. **JSON syntax**: `python3 -c "import json; json.load(open('n0-app.json'))"`
2. **Required fields**: `name`, `slug`, `description`, and at least one service with `image` and `port`
3. **Entrypoint exists**: The `entrypoint` service name must exist in `services`
4. **Image tags**: Every service must have a specific image tag (warn if using `latest`)
5. **Dockerfile builds**: If possible, verify `docker build .` succeeds (Path B only)
6. **Port consistency**: The entrypoint service `port` should match the Dockerfile `EXPOSE` or the upstream container's listening port
7. **Container verification**: For upstream images, confirm the CMD/entrypoint work with the env vars you set — `docker run --rm --entrypoint="" <image> ls` to verify paths exist

```bash
# Validate JSON
python3 -c "
import json, sys
m = json.load(open('n0-app.json'))
errors = []
for f in ['name', 'slug', 'description']:
    if f not in m: errors.append(f'Missing required field: {f}')
if 'services' not in m or not m['services']:
    errors.append('Must have at least one service')
else:
    ep = m.get('entrypoint', 'web')
    if ep not in m['services']:
        errors.append(f'Entrypoint \"{ep}\" not found in services')
    for name, svc in m['services'].items():
        if 'image' not in svc: errors.append(f'Service \"{name}\" missing image')
        if 'port' not in svc: errors.append(f'Service \"{name}\" missing port')
        if svc.get('image','').endswith(':latest'):
            print(f'WARNING: Service \"{name}\" uses :latest tag')
if errors:
    print('ERRORS:'); [print(f'  - {e}') for e in errors]; sys.exit(1)
else:
    print('Manifest is valid')
"
```

### Step 5: Generate Gitea Actions Workflow

Choose the appropriate workflow based on the image strategy:

#### Workflow A: Mirror Upstream Pre-built Images

For apps using upstream pre-built images (e.g., from GHCR or Docker Hub),
create a workflow that pulls the upstream image and re-tags it for the Gitea registry.

**Detect the repo's default branch** and use it in the `branches` trigger (do NOT hardcode `main`).

**CRITICAL: Image name must be lowercase.** `${{ github.repository }}` preserves the original
case (e.g., `clovrlabs/My-App`), but Docker rejects uppercase in image tags. Always define
a lowercase `IMAGE` env var with the hardcoded `{org}/{repo}` in all-lowercase.

```yaml
name: Mirror Upstream Image

on:
  push:
    branches: [DETECT_DEFAULT_BRANCH]
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6am — check for upstream updates

env:
  IMAGE: ${{ vars.REGISTRY }}/LOWERCASE_ORG/LOWERCASE_REPO

jobs:
  mirror-image:
    runs-on: self-hosted
    steps:
      - name: Login to Gitea Registry
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ${{ vars.REGISTRY }} -u "${{ secrets.REGISTRY_USER }}" --password-stdin

      - name: Pull upstream image
        run: |
          docker pull UPSTREAM_REGISTRY/UPSTREAM_IMAGE:UPSTREAM_TAG

      - name: Tag for Gitea registry
        run: |
          docker tag UPSTREAM_REGISTRY/UPSTREAM_IMAGE:UPSTREAM_TAG \
            ${{ env.IMAGE }}:UPSTREAM_TAG
          docker tag UPSTREAM_REGISTRY/UPSTREAM_IMAGE:UPSTREAM_TAG \
            ${{ env.IMAGE }}:latest

      - name: Push to Gitea Container Registry
        run: |
          docker push ${{ env.IMAGE }}:UPSTREAM_TAG
          docker push ${{ env.IMAGE }}:latest

      - name: Cleanup
        if: always()
        run: |
          docker rmi UPSTREAM_REGISTRY/UPSTREAM_IMAGE:UPSTREAM_TAG 2>/dev/null || true
          docker rmi ${{ env.IMAGE }}:UPSTREAM_TAG 2>/dev/null || true
          docker rmi ${{ env.IMAGE }}:latest 2>/dev/null || true
```

Replace `UPSTREAM_REGISTRY/UPSTREAM_IMAGE:UPSTREAM_TAG` with the actual upstream image
(e.g., `ghcr.io/toeverything/affine:stable`), `DETECT_DEFAULT_BRANCH` with the repo's
actual default branch (e.g., `main`, `canary`, `master`), and `LOWERCASE_ORG/LOWERCASE_REPO`
with the all-lowercase org and repo name.

#### Workflow B: Build from Source

For apps with a custom `Dockerfile`, generate a workflow that builds and pushes on every commit.

**Detect the repo's default branch** and use it in the `branches` trigger.

**CRITICAL: Image name must be lowercase.** See note above.

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [DETECT_DEFAULT_BRANCH]

env:
  IMAGE: ${{ vars.REGISTRY }}/LOWERCASE_ORG/LOWERCASE_REPO

jobs:
  build:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Login to Gitea Registry
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ${{ vars.REGISTRY }} -u "${{ secrets.REGISTRY_USER }}" --password-stdin

      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.IMAGE }}:latest \
                       -t ${{ env.IMAGE }}:${{ github.sha }} .
          docker push ${{ env.IMAGE }}:latest
          docker push ${{ env.IMAGE }}:${{ github.sha }}
```

**Notes (both workflows):**
- The runner is `self-hosted` and has Docker available (runs on host, not in container)
- **NEVER use `${{ github.repository }}` directly in Docker image tags** — it preserves the original case (e.g., `clovrlabs/UI-TARS-desktop`) and Docker rejects uppercase. Always use a hardcoded lowercase `IMAGE` env var instead
- Both workflows use `docker login ${{ vars.REGISTRY }}` with org-level secrets `REGISTRY_USER` and `REGISTRY_PASSWORD` to authenticate with this workspace's Gitea container registry. The `REGISTRY` **variable** plus these two **secrets** are auto-provisioned at the org level for every workspace, so any new repo inherits them automatically — never hardcode the registry hostname
- Workflow B uses `actions/checkout@v4` for cloning — do NOT use manual `git clone` with hardcoded runner-internal URLs
- Two tags are pushed: `latest` (for the manifest) and either the commit SHA (for rollback) or the upstream tag (for version tracking)
- **Always detect the repo's default branch** — do not hardcode `main`. Common alternatives: `master`, `canary`, `develop`
- The available runner labels are: `self-hosted` (host mode, has Docker), `ubuntu-latest` (containerized via `node:20-bookworm`), `ubuntu-22.04` (containerized). Use `self-hosted` for any workflow that needs Docker

**Required org-level Actions config (auto-provisioned per workspace):**

| Name | Kind | Description |
|------|------|-------------|
| `REGISTRY` | variable | Host of this workspace's Gitea container registry (e.g. `127.0.0.1:30083`). Use as `${{ vars.REGISTRY }}` in YAML; paste its literal value into `n0-app.json` image fields |
| `REGISTRY_USER` | secret | Username for the Gitea container registry |
| `REGISTRY_PASSWORD` | secret | Password/token (write:package scope) for the Gitea container registry |

These are provisioned automatically at the org level for every workspace, so all repos
inherit them. If they are ever missing, they can be (re)created from **Org Settings →
Actions → Variables / Secrets**, or by re-running the backend `backfill_workspace_gitea_tokens`
provisioning. Confirm the current value of `REGISTRY` with
`GET /api/v1/orgs/{org}/actions/variables/REGISTRY` before writing image paths.

## Common App Patterns

### Single-service web app (most common)
One container serving a web app. The image is built from the repo's Dockerfile.

```json
{
  "name": "My App",
  "slug": "my-app",
  "description": "A web application",
  "entrypoint": "web",
  "services": {
    "web": {
      "image": "127.0.0.1:30083/{org}/{repo}:latest",
      "port": 80,
      "memory_mb": 256
    }
  }
}
```

### Upstream pre-built app (e.g., AFFiNE, Outline, WikiJS)
Using official images from GHCR/Docker Hub with a migration init service.

```json
{
  "name": "My App",
  "slug": "my-app",
  "description": "Self-hosted app using upstream images",
  "badge": "Self-hosted",
  "entrypoint": "web",
  "services": {
    "db": {
      "image": "postgres:16-alpine",
      "port": 5432,
      "env": {
        "POSTGRES_DB": "app",
        "POSTGRES_USER": "app",
        "POSTGRES_PASSWORD": "changeme",
        "POSTGRES_INITDB_ARGS": "--data-checksums"
      },
      "volumes": { "pgdata": "/var/lib/postgresql/data" },
      "memory_mb": 512
    },
    "redis": {
      "image": "redis:7-alpine",
      "port": 6379,
      "volumes": { "redisdata": "/data" },
      "memory_mb": 256
    },
    "migration": {
      "image": "ghcr.io/org/app:stable",
      "port": 3000,
      "env": {
        "DATABASE_URL": "postgresql://app:changeme@db:5432/app",
        "REDIS_HOST": "redis"
      },
      "command": ["sh", "-c", "node ./scripts/migrate.js"],
      "depends_on": ["db", "redis"],
      "memory_mb": 512
    },
    "web": {
      "image": "ghcr.io/org/app:stable",
      "port": 3000,
      "env": {
        "DATABASE_URL": "postgresql://app:changeme@db:5432/app",
        "REDIS_HOST": "redis",
        "APP_URL": "{{APP_URL}}"
      },
      "volumes": {
        "app-data": "/data",
        "app-config": "/config"
      },
      "depends_on": ["db", "redis", "migration"],
      "memory_mb": 1024
    }
  }
}
```

### Web app + database
Frontend/API with a PostgreSQL or MySQL database.

```json
{
  "name": "My App",
  "slug": "my-app",
  "entrypoint": "web",
  "services": {
    "db": {
      "image": "postgres:16-alpine",
      "port": 5432,
      "env": {
        "POSTGRES_DB": "app",
        "POSTGRES_USER": "app",
        "POSTGRES_PASSWORD": "changeme",
        "POSTGRES_INITDB_ARGS": "--data-checksums"
      },
      "volumes": { "pgdata": "/var/lib/postgresql/data" },
      "memory_mb": 512
    },
    "web": {
      "image": "127.0.0.1:30083/{org}/{repo}:latest",
      "port": 3000,
      "env": {
        "DATABASE_URL": "postgres://app:changeme@db:5432/app",
        "APP_URL": "{{APP_URL}}"
      },
      "depends_on": ["db"],
      "memory_mb": 256
    }
  }
}
```

### Web app + Redis
For apps needing caching or session storage.

```json
{
  "services": {
    "redis": {
      "image": "redis:7-alpine",
      "port": 6379,
      "memory_mb": 128
    },
    "web": {
      "image": "127.0.0.1:30083/{org}/{repo}:latest",
      "port": 3000,
      "env": {
        "REDIS_URL": "redis://redis:6379"
      },
      "depends_on": ["redis"],
      "memory_mb": 256
    }
  }
}
```

### Supabase-backed app
Full stack with Supabase (Postgres + Auth + REST API + API Gateway).

```json
{
  "entrypoint": "web",
  "services": {
    "db": {
      "image": "supabase/postgres:15.14.1.107",
      "port": 5432,
      "env": {
        "POSTGRES_DB": "postgres",
        "POSTGRES_USER": "supabase",
        "POSTGRES_PASSWORD": "your-db-password",
        "POSTGRES_INITDB_ARGS": "--data-checksums"
      },
      "volumes": { "pgdata": "/var/lib/postgresql/data" },
      "memory_mb": 512
    },
    "auth": {
      "image": "supabase/gotrue:v2.173.0",
      "port": 9999,
      "env": {
        "GOTRUE_DB_DATABASE_URL": "postgres://supabase:your-db-password@db:5432/postgres?sslmode=disable",
        "GOTRUE_JWT_SECRET": "your-jwt-secret-min-32-chars-long",
        "GOTRUE_SITE_URL": "{{APP_URL}}",
        "GOTRUE_MAILER_AUTOCONFIRM": "true"
      },
      "depends_on": ["db"],
      "memory_mb": 256
    },
    "rest": {
      "image": "postgrest/postgrest:v12.2.8",
      "port": 3000,
      "env": {
        "PGRST_DB_URI": "postgres://supabase:your-db-password@db:5432/postgres",
        "PGRST_DB_ANON_ROLE": "anon",
        "PGRST_JWT_SECRET": "your-jwt-secret-min-32-chars-long"
      },
      "depends_on": ["db"],
      "memory_mb": 256
    },
    "kong": {
      "image": "kong:3.9",
      "port": 8000,
      "env": {
        "KONG_DATABASE": "off",
        "KONG_DECLARATIVE_CONFIG": "/etc/kong/kong.yml"
      },
      "volumes": { "kong-config": "/etc/kong" },
      "config_files": {
        "kong-config": {
          "kong.yml": "_format_version: '2.1'\nservices:\n  - name: auth\n    url: http://auth:9999/\n    routes:\n      - name: auth-route\n        paths: [/auth/v1/]\n        strip_path: true\n  - name: rest\n    url: http://rest:3000/\n    routes:\n      - name: rest-route\n        paths: [/rest/v1/]\n        strip_path: true\n"
        }
      },
      "depends_on": ["auth", "rest"],
      "memory_mb": 256
    },
    "web": {
      "image": "127.0.0.1:30083/{org}/{repo}:latest",
      "port": 80,
      "depends_on": ["kong"],
      "memory_mb": 128
    }
  }
}
```

## Migration / Init Services

Many apps need a one-shot migration step before the main service starts.
Common examples:
- **Prisma**: `npx prisma migrate deploy`
- **Django**: `python manage.py migrate`
- **Rails**: `rails db:migrate`
- **Custom scripts**: `node ./scripts/self-host-predeploy.js`

**Pattern:** Create a `migration` service using the same image as the main app,
with a `command` override that runs the migration script and exits.
The main `web` service should `depends_on` the migration service.

```json
{
  "migration": {
    "image": "same-as-web-image",
    "port": 3000,
    "env": { "...same DB connection vars as web..." },
    "command": ["sh", "-c", "npx prisma migrate deploy"],
    "depends_on": ["db"],
    "memory_mb": 512
  },
  "web": {
    "image": "same-as-web-image",
    "port": 3000,
    "env": { "..." },
    "depends_on": ["db", "migration"],
    "memory_mb": 512
  }
}
```

**Important:** The migration service must have a `port` field even though it
doesn't serve traffic — this is a required field in the schema. Use the same
port as the main app.

## Database Images

### Standard PostgreSQL
For most apps that just need a relational database:

```json
{
  "image": "postgres:16-alpine",
  "port": 5432,
  "env": {
    "POSTGRES_DB": "app",
    "POSTGRES_USER": "app",
    "POSTGRES_PASSWORD": "changeme",
    "POSTGRES_INITDB_ARGS": "--data-checksums"
  },
  "volumes": { "pgdata": "/var/lib/postgresql/data" },
  "memory_mb": 512
}
```

### PostgreSQL with pgvector (AI/embeddings)
For apps that use vector embeddings, similarity search, or AI features
(e.g., AFFiNE, Outline with AI, anything using OpenAI embeddings):

```json
{
  "image": "pgvector/pgvector:pg16",
  "port": 5432,
  "env": {
    "POSTGRES_DB": "app",
    "POSTGRES_USER": "app",
    "POSTGRES_PASSWORD": "changeme",
    "POSTGRES_INITDB_ARGS": "--data-checksums",
    "POSTGRES_HOST_AUTH_METHOD": "trust"
  },
  "volumes": { "pgdata": "/var/lib/postgresql/data" },
  "memory_mb": 512
}
```

**When to use pgvector:** If the app's dependencies include pgvector, vector
extensions, or if the Prisma schema / SQL migrations reference `vector` types.

### PostgreSQL environment variables reference

| Variable | Purpose | Example |
|----------|---------|---------|
| `POSTGRES_DB` | Database name to create | `app` |
| `POSTGRES_USER` | Superuser username | `app` |
| `POSTGRES_PASSWORD` | Superuser password | `changeme` |
| `POSTGRES_INITDB_ARGS` | Extra args for `initdb` | `--data-checksums` |
| `POSTGRES_HOST_AUTH_METHOD` | Auth method for local connections | `trust` or `scram-sha-256` |

## Security Notes

- **Never hardcode real secrets** in the manifest. Use empty placeholder values -- real secrets are injected from Vault at deploy time (see "Secrets Management with Vault" below).
- Containers run with dangerous capabilities dropped: `NET_RAW`, `SYS_ADMIN`, `MKNOD`, `AUDIT_WRITE`
- Containers run with `no-new-privileges:true`
- `extra_hosts` from manifests are blocked (no access to host services)
- Each multi-container app gets its own isolated Docker network

## Secrets Management with Vault

**N0 uses HashiCorp Vault for secrets management.** App secrets should NOT be hardcoded in manifests. Instead, use empty placeholder values in the manifest and store the real secrets in Vault.

### How It Works

At deploy time, the AppManager merges environment variables from three sources (in priority order):

1. **Manifest `env`** -- default/placeholder values from `n0-app.json`
2. **`env_overrides`** -- per-instance overrides set via the API
3. **Vault secrets** (highest priority) -- loaded from `secret/clovr/apps/{app-type}` and `secret/clovr/apps/{app-type}/{service-name}`

Vault secrets override everything else, so manifests can ship with empty or placeholder values for sensitive fields.

### Vault KV Paths

```
secret/clovr/apps/{slug}                  # Shared secrets (all services)
secret/clovr/apps/{slug}/{service-name}   # Per-service secrets
```

**Examples:**
```
secret/clovr/apps/qwen-chat               # OPENAI_API_KEY, WEBUI_SECRET_KEY
secret/clovr/apps/affine/db               # POSTGRES_PASSWORD
secret/clovr/apps/crm-pro                 # Shared JWT secret
secret/clovr/apps/crm-pro/db              # POSTGRES_PASSWORD
secret/clovr/apps/crm-pro/auth            # GOTRUE_JWT_SECRET, GOTRUE_DB_DATABASE_URL
```

### Writing Manifests with Vault

**Preferred: Dict format with metadata** -- declares secrets with auto-generate capability:
```json
{
  "env": {
    "DATABASE_URL": "postgres://app:@db:5432/app",
    "POSTGRES_PASSWORD": {
      "secret": true,
      "generate": true,
      "label": "Database password"
    },
    "API_KEY": {
      "secret": true,
      "generate": true,
      "label": "API key"
    },
    "JWT_SECRET": {
      "secret": true,
      "generate": true,
      "label": "JWT signing secret"
    }
  }
}
```

**Also supported: Empty string placeholders** -- legacy format, still works with auto-detection:
```json
{
  "env": {
    "DATABASE_URL": "postgres://app:@db:5432/app",
    "POSTGRES_PASSWORD": "",
    "API_KEY": "",
    "JWT_SECRET": ""
  }
}
```

Keys matching `PASSWORD`, `SECRET`, `TOKEN`, or `API_KEY` patterns with empty or placeholder values (`changeme`, `secret`, `replace_me`) are automatically detected as generatable secrets.

**DON'T:** Hardcode real secrets in manifests:
```json
{
  "env": {
    "POSTGRES_PASSWORD": "my-real-password-123"
  }
}
```

### Managing Secrets via the UI

Users can manage secrets directly from the N0 web interface -- no SSH or Vault CLI access needed:

1. Click **Secrets** on an installed app (visible to admins in both grid and list views)
2. The modal shows all secret fields grouped by service, with **Set** badges for already-configured secrets
3. Type values manually or click **Generate** per field to create a 32-character random password
4. Click **Auto-generate all missing secrets** to fill in all empty generatable fields at once
5. Click **Save & Redeploy** -- values are saved to Vault and the app is automatically redeployed

### Storing Secrets via Vault CLI

For advanced use cases, secrets can also be set via the Vault CLI on the server:

```bash
# Set VAULT_ADDR and authenticate
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=<root-token>

# Store app-level secrets (shared across all services)
vault kv put secret/clovr/apps/my-app \
  API_KEY="real-api-key" \
  JWT_SECRET="real-jwt-secret"

# Store per-service secrets (e.g., database password)
vault kv put secret/clovr/apps/my-app/db \
  POSTGRES_PASSWORD="real-db-password"

# Verify
vault kv get secret/clovr/apps/my-app
```

### Multi-Container Apps with Shared Secrets

For apps where multiple services need the same secret (e.g., a JWT secret shared between auth and API services), store it at the app level:

```bash
# Shared secret -- available to all services
vault kv put secret/clovr/apps/my-app \
  JWT_SECRET="shared-jwt-secret" \
  DB_PASSWORD="shared-db-password"

# Per-service secrets that include the shared ones in connection strings
vault kv put secret/clovr/apps/my-app/auth \
  GOTRUE_DB_DATABASE_URL="postgres://admin:shared-db-password@db:5432/postgres"

vault kv put secret/clovr/apps/my-app/db \
  POSTGRES_PASSWORD="shared-db-password"
```

### Auto-Generation at Deploy Time

At deploy time, secrets matching PASSWORD/SECRET/TOKEN/API_KEY patterns that have empty or placeholder values are automatically generated as 32-character random strings and stored back to Vault. This ensures apps work out of the box without manual secret configuration.

### Cleanup on Uninstall

When an app is uninstalled, its Vault secrets are automatically cleaned up:
- `secret/clovr/apps/{slug}` (app-level secrets)
- `secret/clovr/apps/{slug}/{service}` (per-service secrets for each service in the manifest)

This prevents orphaned secrets from accumulating in Vault.

### Checklist for Secrets

- [ ] No real secrets in `n0-app.json` (use empty strings or dict format with `"secret": true`)
- [ ] Secret fields use dict format where possible: `{ "secret": true, "generate": true, "label": "..." }`
- [ ] All sensitive env vars stored in Vault at `secret/clovr/apps/{slug}`
- [ ] Per-service secrets stored at `secret/clovr/apps/{slug}/{service-name}`
- [ ] Database passwords are consistent across services that share them
- [ ] Connection strings in Vault use the correct inter-service hostnames (Docker DNS names like `db`, `redis`)

## SSO / Authentication Integration

**N0 provides automatic Single Sign-On (SSO) for all hosted apps.** Every request to a hosted app goes through Caddy's `forward_auth`, which validates the user's session and injects identity headers into the upstream request. Apps that support reverse-proxy authentication can use these headers to automatically log users in — no separate login required.

### How It Works

1. User visits `https://myapp.apps.privateprompt.tech`
2. Caddy intercepts and sends a sub-request to Django's `/apps/auth-check/` endpoint
3. Django checks the user's **session cookie** (shared across all `*.privateprompt.tech` subdomains)
4. If authenticated and authorized → Caddy injects these headers into the request to the container:

| Header | Value | Example |
|--------|-------|---------|
| `X-Webauth-User` | Email prefix (username) | `rasmus` |
| `X-Webauth-Email` | Full email address | `rasmus@clovrlabs.com` |
| `X-Webauth-Fullname` | Display name | `Rasmus Schlunsen` |

5. The app reads these headers and auto-creates/logs in the user — **zero-click SSO**

### Configuring Apps for SSO

If the app you're deploying supports **reverse proxy authentication**, enable it via environment variables. The exact env vars depend on the app:

#### Gitea
```json
{
  "env": {
    "GITEA__service__ENABLE_REVERSE_PROXY_AUTHENTICATION": "true",
    "GITEA__service__ENABLE_REVERSE_PROXY_AUTO_REGISTRATION": "true",
    "GITEA__service__ENABLE_REVERSE_PROXY_EMAIL": "true",
    "GITEA__service__ENABLE_REVERSE_PROXY_FULL_NAME": "true",
    "GITEA__security__REVERSE_PROXY_LIMIT": "1",
    "GITEA__security__REVERSE_PROXY_TRUSTED_PROXIES": "127.0.0.1",
    "GITEA__service__DISABLE_REGISTRATION": "true",
    "GITEA__service__REQUIRE_SIGNIN_VIEW": "true"
  }
}
```

#### Outline
```json
{
  "env": {
    "AUTHENTICATION_PROVIDER": "header",
    "AUTH_HEADER_EMAIL": "X-Webauth-Email",
    "AUTH_HEADER_NAME": "X-Webauth-Fullname"
  }
}
```

#### Grafana
```json
{
  "env": {
    "GF_AUTH_PROXY_ENABLED": "true",
    "GF_AUTH_PROXY_HEADER_NAME": "X-Webauth-User",
    "GF_AUTH_PROXY_HEADER_PROPERTY": "username",
    "GF_AUTH_PROXY_AUTO_SIGN_UP": "true",
    "GF_AUTH_PROXY_HEADERS": "Email:X-Webauth-Email Name:X-Webauth-Fullname"
  }
}
```

#### Backstage / Custom Apps
For custom apps, read the headers in your middleware:
```python
# Django middleware example
class ReverseProxyAuthMiddleware:
    def __call__(self, request):
        email = request.META.get("HTTP_X_WEBAUTH_EMAIL")
        username = request.META.get("HTTP_X_WEBAUTH_USER")
        fullname = request.META.get("HTTP_X_WEBAUTH_FULLNAME")
        if email:
            user, _ = User.objects.get_or_create(
                email=email,
                defaults={"username": username, "full_name": fullname}
            )
            request.user = user
        return self.get_response(request)
```

```javascript
// Express.js middleware example
app.use((req, res, next) => {
  const email = req.headers['x-webauth-email'];
  const username = req.headers['x-webauth-user'];
  const fullname = req.headers['x-webauth-fullname'];
  if (email) {
    req.user = { email, username, fullname };
  }
  next();
});
```

### When to Configure SSO

- **Apps with built-in reverse proxy auth support** (Gitea, Grafana, Outline, WikiJS, etc.): Set the appropriate env vars in the manifest
- **Custom apps**: Add middleware to read `X-Webauth-*` headers and auto-create/authenticate users
- **Apps with their own auth only** (e.g., Supabase, n8n): Skip SSO config — N0 still handles access control at the Caddy layer, but the app manages its own internal user sessions

### Access Control Levels

N0 enforces access control at the Caddy layer *before* the request reaches your app:

| Level | Who can access |
|-------|---------------|
| `PUBLIC` | Anyone (no authentication required) |
| `WORKSPACE` | Any member of the workspace |
| `RESTRICTED` | Only specific allowed users + workspace admins |

This is configured per-app in the N0 admin UI, not in the manifest.

### Public Path Bypass

Some app endpoints need to be accessible without authentication (webhooks, health checks, API callbacks). Configure these in the N0 admin UI as **public paths**:

- `/health` — exact match
- `/api/webhook/*` — prefix match (everything under `/api/webhook/`)
- `/callback` — exact match

These paths bypass both Caddy forward_auth and Django access checks.

### API Token Passthrough

Requests to `/api/*` or `/graphql` paths carrying an `Authorization: Bearer *` header bypass forward_auth entirely. This allows external API clients (CI/CD, CLI tools, mobile apps) to authenticate directly with the hosted app using the app's own token system, without needing a N0 session cookie.

### SSO Checklist

When generating a manifest for an app that supports reverse proxy auth:

- [ ] Enable reverse proxy / header-based authentication in the app's env vars
- [ ] Set the trusted proxy to `127.0.0.1` (the Caddy reverse proxy)
- [ ] Map `X-Webauth-User`, `X-Webauth-Email`, and `X-Webauth-Fullname` to the app's expected header names
- [ ] Enable auto-registration so users are created on first visit
- [ ] Disable the app's built-in registration page (users come through N0)
- [ ] Disable the app's built-in login page if possible (users are already authenticated)
- [ ] Document any public paths that should bypass auth (webhooks, health checks)


## App-Published Connectors

Apps can **publish connectors** that workspace users and agents can use. When an app declares connectors in its manifest, they appear in the "App Connectors" tab in the workspace settings. Users can enable them with one click, and agents can make API calls to the app through a platform proxy with automatic user identity injection.

### How It Works

1. App declares `connectors` in its `n0-app.json` manifest
2. On deploy, N0 creates `ConnectorDefinition` records for each declared connector
3. Users see them in **Connectors → App Connectors** tab and click **Enable**
4. Agents call the app's API through a proxy — N0 injects `X-Webauth-*` headers with the granting user's identity
5. The app receives the same identity headers as browser requests — **zero app-side auth code needed**

### Auth Types

#### `platform_identity` (default, recommended)

No credentials are stored or exchanged. The platform proxies agent requests with the user's identity headers. The app just reads `X-Webauth-*` headers (same as browser access via SSO).

**This is the preferred auth type** — it's zero-config for the app and eliminates credential management entirely.

#### `api_key` (fallback)

For apps that have their own authentication system and can't use reverse proxy auth. Users enter an API key manually, which is encrypted and stored.

### Connector Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `slug` | Yes | string | Unique identifier within the app (e.g., `my-api`). Namespaced as `{subdomain}--{slug}` internally |
| `name` | Yes | string | Display name (e.g., `My App API`) |
| `description` | No | string | What the connector provides (shown in UI) |
| `auth_type` | No | string | `platform_identity` (default) or `api_key` |
| `icon_emoji` | No | string | Display emoji (default: `📦`) |
| `base_path` | No | string | API base path within the app (e.g., `/api/v1`). Must start with `/`, no path traversal |
| `scopes` | No | string[] | Available scopes (informational, displayed in UI) |
| `help_text` | No | string | Help text shown when connecting (useful for `api_key` type) |
| `help_url_path` | No | string | Relative path to help page within the app (must start with `/`) |

### Example: Platform Identity Connector

This is the most common pattern. The app just reads `X-Webauth-*` headers:

```json
{
  "name": "Inventory Manager",
  "slug": "inventory",
  "services": { "...": "..." },
  "connectors": [
    {
      "slug": "inventory-api",
      "name": "Inventory API",
      "description": "Product catalog and stock management",
      "auth_type": "platform_identity",
      "icon_emoji": "📦",
      "base_path": "/api/v1",
      "scopes": ["products:read", "products:write", "stock:read", "stock:write"]
    }
  ]
}
```

The app's API endpoints just need to read the identity headers:

```python
# FastAPI example
from fastapi import Request

@app.get("/api/v1/products")
def list_products(request: Request):
    user_email = request.headers.get("X-Webauth-Email")
    username = request.headers.get("X-Webauth-User")
    # Use the identity to filter/authorize...
```

```javascript
// Express.js example
app.get('/api/v1/products', (req, res) => {
  const userEmail = req.headers['x-webauth-email'];
  const username = req.headers['x-webauth-user'];
  // Use the identity to filter/authorize...
});
```

### Example: API Key Connector

For apps with their own token system:

```json
{
  "connectors": [
    {
      "slug": "legacy-api",
      "name": "Legacy API",
      "description": "Access via API key",
      "auth_type": "api_key",
      "icon_emoji": "🔑",
      "base_path": "/api",
      "help_text": "Go to Settings > API Keys to generate a key",
      "help_url_path": "/settings/api-keys"
    }
  ]
}
```

### Multiple Connectors

An app can publish multiple connectors (e.g., separate read/write APIs):

```json
{
  "connectors": [
    {
      "slug": "read-api",
      "name": "Read API",
      "description": "Read-only access to data",
      "base_path": "/api/v1",
      "scopes": ["data:read"]
    },
    {
      "slug": "admin-api",
      "name": "Admin API",
      "description": "Full admin access",
      "base_path": "/api/admin",
      "scopes": ["data:read", "data:write", "users:manage"]
    }
  ]
}
```

### Detecting API Routes

When analyzing a codebase, look for API routes to auto-generate the `connectors` section:

- **Express/Fastify**: `app.get('/api/...')`, `router.post(...)`, route files in `routes/`
- **Django**: `urlpatterns`, `path('api/...')`, `@api_view`
- **FastAPI**: `@app.get('/api/...')`, `APIRouter`
- **Flask**: `@app.route('/api/...')`
- **Go**: `http.HandleFunc`, `mux.Handle`, `gin.GET`

Map route patterns to scopes:
- `GET /api/contacts` → `contacts:read`
- `POST /api/contacts` → `contacts:write`
- `DELETE /api/contacts/:id` → `contacts:delete`

### Connector Lifecycle

- **Deploy/Redeploy**: Connectors are synced from the manifest. New ones are created, removed ones are deleted, existing ones are updated.
- **Stop**: Connectors are deactivated (show "App not running" in UI). Permission records are preserved.
- **Start**: Connectors are reactivated immediately.
- **Uninstall**: All connectors and their permission records are cascade-deleted.

### Connectors Checklist

- [ ] `connectors` array is valid (each entry has at least `slug` and `name`)
- [ ] `auth_type` is `platform_identity` (default) or `api_key`
- [ ] `base_path` starts with `/` and has no path traversal (`..`)
- [ ] `help_url_path` starts with `/` (if provided)
- [ ] App reads `X-Webauth-*` headers for `platform_identity` connectors (same as SSO)
- [ ] Scopes are descriptive and follow `resource:action` pattern
- [ ] For `api_key` connectors: `help_text` explains how to get a key

## Icon Color Palette

Common Tailwind color options for the `icon` field:

| Color | bg | color |
|-------|-----|-------|
| Indigo | `bg-indigo-500/20` | `text-indigo-400` |
| Green | `bg-green-500/20` | `text-green-400` |
| Blue | `bg-blue-500/20` | `text-blue-400` |
| Purple | `bg-purple-500/20` | `text-purple-400` |
| Orange | `bg-orange-500/20` | `text-orange-400` |
| Red | `bg-red-500/20` | `text-red-400` |
| Sky | `bg-sky-500/20` | `text-sky-400` |
| Amber | `bg-amber-500/20` | `text-amber-400` |
| Rose | `bg-rose-500/20` | `text-rose-400` |
| Teal | `bg-teal-500/20` | `text-teal-400` |

## Checklist

Before finalizing, verify:

- [ ] `n0-app.json` is valid JSON
- [ ] All required fields present (`name`, `slug`, `description`, `services`)
- [ ] Entrypoint service exists and has correct `port`
- [ ] All services have `image` with specific tag
- [ ] **Custom images use the workspace registry path** (`<REGISTRY>/{org}/{repo}:tag`, where `<REGISTRY>` is the literal value of the `REGISTRY` org variable, e.g. `127.0.0.1:30083`) — NEVER bare names like `my-app:latest`
- [ ] **Upstream images use the full registry path** (e.g., `ghcr.io/org/app:stable`) — NOT just the image name
- [ ] `.gitea/workflows/build-and-push.yml` exists (build from source OR mirror upstream)
- [ ] **Workflow uses a hardcoded lowercase `IMAGE` env var** — NEVER use `${{ github.repository }}` in Docker tags (it preserves uppercase and Docker rejects it)
- [ ] **Workflow branch trigger matches the repo's actual default branch** (not hardcoded `main`)
- [ ] Database services have `volumes` for data persistence
- [ ] Database services include `POSTGRES_INITDB_ARGS: --data-checksums` for PostgreSQL
- [ ] `depends_on` ordering is correct (databases -> migration -> app)
- [ ] Migration/init services are defined for apps that need DB setup before starting
- [ ] Environment variables use `{{APP_URL}}` / `{{APP_DOMAIN}}` where needed
- [ ] **Env vars and paths were verified against the upstream self-host compose** or container inspection — NEVER guessed
- [ ] Dockerfile builds successfully (if custom image needed)
- [ ] `.dockerignore` excludes `node_modules`, `.git`, etc. (if custom Dockerfile)
- [ ] No real secrets in the manifest (use dict format or empty placeholders -- secrets are managed via UI or Vault)
- [ ] Memory limits are reasonable for each service
- [ ] Health check endpoints use `0.0.0.0` not `localhost` (Alpine musl resolves localhost to IPv6 `::1`)
- [ ] Cross-app references (`{{APP:<slug>:<field>}}`) point to apps that exist
- [ ] `access_level` set in manifest if the app should be public
- [ ] If the app has API routes, `connectors` section is included in the manifest
- [ ] Connector `base_path` starts with `/` and matches the app's actual API prefix
- [ ] App reads `X-Webauth-*` headers for `platform_identity` connectors

## Deploy Validation

N0 runs 7 automated pre-deploy checks before launching containers. When writing manifests, be aware of these checks so your app passes validation:

| Check | Severity | What It Catches |
|-------|----------|-----------------|
| Image pullable | error | Image doesn't exist in registry or tag is wrong |
| Stale Vault keys | warning | Vault has secrets for env vars no longer in the manifest |
| Health check localhost | warning | Health check commands using `localhost` instead of `0.0.0.0` (fails on Alpine musl) |
| Access level mismatch | warning | Manifest says `public` but DB still has `workspace` |
| Cross-app refs | warning | `{{APP:<slug>:<field>}}` references a non-running app |
| CORS wildcard | warning | `CORS_ORIGIN=*` may be rejected by production apps |
| Missing env values | warning | Required env vars are empty after all sources merged |

You can check validation before deploying via the API:
```
GET /api/hosted-apps/{app_id}/validate
```

Returns:
```json
{
  "issues": [
    {"level": "warning", "code": "stale_vault_keys", "message": "Vault has keys not in manifest: OLD_KEY"}
  ],
  "error_count": 0,
  "warning_count": 1,
  "deploy_ok": true
}
```

## Health Monitoring

N0 monitors Docker container health every 60 seconds for all running apps. The health status is stored per-service on the `HostedApp` model and exposed in the API.

### How It Works

1. A background task queries all apps with `status=RUNNING`
2. For each app, it inspects `docker inspect` → `State.Health.Status` for each container
3. If a container has a Docker `HEALTHCHECK`, the status is reported (e.g., `healthy`, `unhealthy`, `starting`)
4. If a container has no `HEALTHCHECK`, the status is reported as `running` (based on container state)
5. If a container is not found, the status is `not_found`
6. If ALL containers are `not_found`, the app is automatically marked `ERRORED`

### Health Check Best Practices

When writing Dockerfiles or using upstream images with health checks:

```dockerfile
# Use 0.0.0.0, NOT localhost (Alpine musl IPv6 issue)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://0.0.0.0:3000/health || exit 1
```

**Common pitfall:** `wget -qO- http://localhost:3000/health` fails on Alpine because musl libc resolves `localhost` to `::1` (IPv6) but most apps only listen on `0.0.0.0` (IPv4). Always use `http://0.0.0.0:<port>` instead.

## Deploying to the Platform (API Flow)

After generating the manifest and pushing code to Gitea, the app must be **imported** and then **deployed** via the N0 API. This is a two-step process:

### Prerequisites

1. **Workspace admin JWT token** — only workspace admins can import and deploy apps
2. **Code + n0-app.json pushed to Gitea** — the repo must exist in the workspace's Gitea instance
3. **Gitea Actions build completed** — the Docker image must be in the registry before deploy

### Step 1: Import App Definition

Register the app in the workspace catalog by pointing at the Gitea repo:

```bash
# Login to get JWT token (must be a workspace admin)
TOKEN=$(curl -s -X POST https://app.privateprompt.tech/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"..."}' | \
  python3 -c "import json,sys; print(json.load(sys.stdin)['data']['tokens']['access_token'])")

# Import app definition from Gitea repo
curl -s -X POST "https://app.privateprompt.tech/api/v1/workspaces/${WS_ID}/apps/definitions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"repo_url": "clovrlabs/my-app"}'
```

This fetches `n0-app.json` from the repo, validates it, and creates an `AppDefinition` record. The `repo_url` accepts:
- Short form: `org/repo` (uses workspace's own Gitea)
- Full URL: `https://gitea-clovrlabs.apps.privateprompt.tech/org/repo`

### Step 2: Deploy App Instance

```bash
curl -s -X POST "https://app.privateprompt.tech/api/v1/workspaces/${WS_ID}/apps/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"app_type": "my-app-slug", "subdomain": "my-app"}'
```

- `app_type` must match the `slug` in n0-app.json (and the registered AppDefinition)
- `subdomain` becomes `https://{subdomain}.apps.privateprompt.tech`
- Optional: `"access_level": "public"` or `"env_overrides": {"KEY": "val"}`

The deploy is async — a background Huey task:
1. Pulls/mirrors the Docker image into the k3s cluster
2. Creates Kubernetes Deployments + Services in `n0-{workspace-slug}` namespace
3. Allocates a NodePort for the entrypoint service
4. Registers a Caddy reverse-proxy route
5. Creates Cloudflare DNS record
6. Sets status to `running`

### Step 3: Redeploy (after code changes)

```bash
curl -s -X POST "https://app.privateprompt.tech/api/v1/workspaces/${WS_ID}/apps/${APP_ID}/redeploy" \
  -H "Authorization: Bearer $TOKEN"
```

### Other Operations

```bash
# Stop app
curl -X POST ".../apps/${APP_ID}/stop" -H "Authorization: Bearer $TOKEN"

# Start app
curl -X POST ".../apps/${APP_ID}/start" -H "Authorization: Bearer $TOKEN"

# Delete/uninstall app
curl -X DELETE ".../apps/${APP_ID}" -H "Authorization: Bearer $TOKEN"

# Pre-deploy validation
curl ".../apps/${APP_ID}/validate" -H "Authorization: Bearer $TOKEN"
```

### Image Mirroring (How K8s Pulls Images)

The K8sAppManager handles image sources automatically:

| Image Pattern | Behavior |
|--------------|----------|
| `127.0.0.1:{port}/{org}/{repo}:{tag}` | Workspace Gitea registry — mirrored into in-cluster registry via skopeo Job |
| `postgres:16-alpine`, `redis:7-alpine` | Public Docker Hub — pulled directly by containerd |
| `ghcr.io/org/app:stable` | External registry — pulled directly by containerd |
| `localhost:5000/{path}` | Already in in-cluster registry — used as-is |

**Important:** Images from the workspace Gitea registry use `127.0.0.1:{gitea_http_port}` as the host (NOT the external hostname). The runner pushes via loopback (hostNetwork), and the K8sAppManager recognizes the `127.0.0.1:` prefix to trigger skopeo mirroring.

### Common Deployment Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `AppDefinition matching query does not exist` | No AppDefinition for this `app_type` slug | Import the app first via `POST /definitions` |
| `Only workspace admins can deploy apps` | User is not a workspace admin | Use an admin account |
| `Unknown app type` | The `app_type` doesn't match any built-in or AppDefinition slug | Check the slug in n0-app.json |
| Image pull failure | Image not in registry or wrong tag | Check Gitea Actions build succeeded; verify `REGISTRY_USER`/`REGISTRY_PASSWORD` org secrets |
| Build push failure | Stale or missing registry credentials | Re-provision org-level Actions secrets (see below) |

### Troubleshooting Registry Credentials

If Gitea Actions builds succeed but `docker push` fails, the org-level Actions secrets may be stale:

```bash
# On the staging server (SSH required):
# 1. Generate a fresh registry push token
kubectl -n n0-clovrlabs exec deployment/gitea -- \
  gitea admin user generate-access-token \
  --username n0-system-admin \
  --token-name registry-push-$(date +%s) \
  --scopes write:package,read:package --raw

# 2. Update org secrets via Gitea API
ADMIN_TOKEN="<gitea-admin-token>"
curl -X PUT -H "Authorization: token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "http://127.0.0.1:32102/api/v1/orgs/clovrlabs/actions/secrets/REGISTRY_PASSWORD" \
  -d '{"data": "<new-token>"}'
```

## Complete Local Dev → Deploy Workflow

Here's the full workflow for developing an app locally and deploying it to N0:

### 1. Create a Gitea Repo

```bash
GITEA_TOKEN="<your-gitea-personal-access-token>"
GITEA_HOST="gitea-clovrlabs.apps.privateprompt.tech"

curl -s -X POST "https://${GITEA_HOST}/api/v1/orgs/clovrlabs/repos" \
  -H "Authorization: token $GITEA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-app", "auto_init": true, "private": false}'
```

### 2. Develop Locally

Build your app with any framework. Add these files to the repo root:
- `n0-app.json` — N0 manifest (see schema above)
- `Dockerfile` — multi-stage build for production
- `.dockerignore` — exclude `node_modules`, `.git`, etc.
- `.gitea/workflows/build-and-push.yml` — CI workflow

### 3. Look Up the REGISTRY Value

Before writing the manifest, look up the org-level `REGISTRY` variable:

```bash
curl -s -H "Authorization: token $GITEA_TOKEN" \
  "https://${GITEA_HOST}/api/v1/orgs/clovrlabs/actions/variables/REGISTRY" | \
  python3 -c "import json,sys; print(json.load(sys.stdin)['value'])"
```

Use this literal value (e.g., `127.0.0.1:32102`) in both `n0-app.json` image fields AND the workflow `IMAGE` env var.

### 4. Push to Gitea

```bash
git init && git add -A && git commit -m "Initial commit"
git remote add origin "https://<username>:${GITEA_TOKEN}@${GITEA_HOST}/clovrlabs/my-app.git"
git push -u origin main --force
```

### 5. Wait for CI Build

```bash
# Poll until the Gitea Actions build completes
curl -s -H "Authorization: token $GITEA_TOKEN" \
  "https://${GITEA_HOST}/api/v1/repos/clovrlabs/my-app/actions/runs" | \
  python3 -c "import json,sys; r=json.load(sys.stdin)['workflow_runs'][0]; print(f'status={r[\"status\"]} conclusion={r[\"conclusion\"]}')"
```

### 6. Import + Deploy via API

```bash
N0_TOKEN="<admin-jwt-token>"
WS_ID="<workspace-uuid>"
API="https://app.privateprompt.tech/api/v1"

# Import app definition
curl -s -X POST "$API/workspaces/$WS_ID/apps/definitions" \
  -H "Authorization: Bearer $N0_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"repo_url": "clovrlabs/my-app"}'

# Deploy
curl -s -X POST "$API/workspaces/$WS_ID/apps/" \
  -H "Authorization: Bearer $N0_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"app_type": "my-app", "subdomain": "my-app"}'
```

### 7. Verify

```bash
# Check app status
curl -s "$API/workspaces/$WS_ID/apps/" \
  -H "Authorization: Bearer $N0_TOKEN" | \
  python3 -c "import json,sys; apps=json.load(sys.stdin)['data']; [print(f'{a[\"subdomain\"]}: {a[\"status\"]}') for a in apps if a['subdomain']=='my-app']"

# Visit the app
open "https://my-app.apps.privateprompt.tech"
```

### Getting Credentials for Local Development

**Gitea personal access token** (for git push):
- Visit `https://gitea-clovrlabs.apps.privateprompt.tech/-/user/settings/applications`
- Click "Generate New Token" → select scopes → copy token
- Or via API (requires admin):
  ```bash
  kubectl -n n0-clovrlabs exec deployment/gitea -- \
    gitea admin user generate-access-token \
    --username <your-username> --token-name dev --scopes all --raw
  ```

**Supabase credentials** (for database access):
- Visible in the N0 web UI under **Connectors** page
- Or via the API (workspace connection):
  ```bash
  curl -s "$API/workspaces/$WS_ID/connectors/" \
    -H "Authorization: Bearer $N0_TOKEN" | \
    python3 -c "import json,sys; conns=json.load(sys.stdin)['data']['connections']; [print(f'{c[\"connector_slug\"]}: {c.get(\"extra_config\",{})}') for c in conns if c['connector_slug']=='supabase']"
  ```

**N0 API JWT token** (for deploying):
```bash
curl -s -X POST "$API/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"..."}' | \
  python3 -c "import json,sys; print(json.load(sys.stdin)['data']['tokens']['access_token'])"
```

