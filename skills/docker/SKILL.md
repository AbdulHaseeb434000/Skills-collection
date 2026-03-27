---
name: docker
description: >
  Expert Docker and container engineering guide covering Dockerfiles, Docker Compose,
  image optimization, security hardening, networking, and CI/CD integration.
  Use this skill whenever the user wants to: write or improve a Dockerfile, set up
  Docker Compose services, optimize image size or build speed, containerize an
  application (Node.js, Python, Go, Java, or any stack), fix Docker networking or
  volume issues, harden container security, push/pull images from registries, debug
  running containers, set up Docker in CI/CD pipelines, or ask any question about
  Docker best practices. Trigger even for vague requests like "containerize my app",
  "make my Docker build faster", or "my container keeps crashing" — these all benefit
  from this skill.
---

# Docker Engineering Skill

You are a container expert. Your job is to help users write correct, optimized, and
secure Docker configurations. Always write production-ready code — not tutorial
scaffolding. When in doubt, optimize for image size, build speed, and security.

## Quick Reference

Read `references/cli.md` for the full Docker & Compose CLI command reference.
Read `references/stacks.md` for language-specific Dockerfile patterns (Node.js, Python, Go, Java).

---

## Dockerfile Fundamentals

### Layer & Cache Optimization

Docker builds images layer by layer. Each instruction creates a layer. Layers are
cached — if nothing changed, Docker reuses them. **Exploit this by ordering instructions
from least-frequently-changed to most-frequently-changed.**

```dockerfile
# BAD: code copy invalidates cache before deps install
COPY . .
RUN pip install -r requirements.txt

# GOOD: deps install cached separately from code
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

**Key rule:** Copy only what you need before each install step. Copy everything else last.

### Multi-Stage Builds

Use multi-stage builds to separate build-time dependencies from the final image.
The final image only contains what's needed to run — not compilers, build tools, or source code.

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app/server .

# Stage 2: Runtime (tiny final image)
FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

For interpreted languages (Python, Node.js), multi-stage still helps by removing
dev dependencies, build tools, and test files from the final image.

### .dockerignore

Always create a `.dockerignore`. Without it, `COPY . .` sends your entire repo
(including `node_modules`, `.git`, test fixtures) to the Docker daemon on every build.

```
.git
.gitignore
node_modules
__pycache__
*.pyc
*.pyo
.pytest_cache
.coverage
dist
build
*.log
.env
.env.*
docker-compose*.yml
Dockerfile*
README.md
tests/
docs/
```

### Package Install Best Practices

**Never use cache for package managers in Docker** — it bloats the layer with index files
that aren't needed at runtime.

```dockerfile
# Alpine (apk)
RUN apk add --no-cache curl git

# Debian/Ubuntu (apt)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
  && rm -rf /var/lib/apt/lists/*

# Python (pip)
RUN pip install --no-cache-dir -r requirements.txt

# Node.js (npm)
RUN npm ci --only=production
```

The `--no-install-recommends` flag for apt and `--no-cache-dir` for pip prevent
pulling in extra packages you didn't ask for.

### BuildKit Cache Mounts (Advanced)

For local development builds where you want fast iteration without bloating layers,
use BuildKit cache mounts. These cache package data on the host between builds
without including it in the image layer.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim

# pip cache mount — fast repeated builds, zero layer bloat
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# apt cache mount
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y --no-install-recommends curl
```

Enable BuildKit: `DOCKER_BUILDKIT=1 docker build .` or set in daemon config.

### ARG vs ENV

- `ARG` — build-time only. Not available in the running container. Use for build
  configuration (version pins, feature flags during build).
- `ENV` — persists into the running container. Use for runtime config defaults.
- Never put secrets in either — they appear in `docker inspect` and image history.

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

# Runtime defaults (overridable at `docker run` time)
ENV NODE_ENV=production \
    PORT=3000
```

**Important:** `ARG` before `FROM` only scopes to the `FROM` line. Re-declare after
`FROM` if you need it in build steps.

### Base Image Selection

| Use case | Base image | Why |
|---|---|---|
| Smallest possible | `scratch`, `distroless` | No shell, no OS — attack surface ~zero |
| Minimal Linux | `alpine:3.x` | ~5MB, `apk`, good for most things |
| Compatibility needed | `debian:bookworm-slim` | Wider glibc compat, larger (~80MB) |
| Language-specific | `python:3.12-slim`, `node:20-alpine` | Pre-installed runtime |

Avoid `latest` tags in production — pin to a specific version so builds are reproducible.

---

## Docker Compose

### Service Definition Template

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_VERSION: "20"
    image: myapp:latest
    container_name: myapp
    restart: unless-stopped
    ports:
      - "3000:3000"          # host:container
    environment:
      NODE_ENV: production
    env_file:
      - .env                 # load from file (don't commit this)
    volumes:
      - app_data:/data       # named volume
      - ./config:/app/config:ro  # bind mount, read-only
    networks:
      - backend
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  app_data:
  db_data:

networks:
  backend:
    driver: bridge
```

### Key Compose Concepts

**`depends_on` with `condition`** — ensures services start in the right order and
only after dependencies are actually ready (not just started). Use `service_healthy`
when a healthcheck is defined, `service_started` otherwise.

**`restart: unless-stopped`** — restarts on crash but respects manual `docker stop`.
Use `always` only if you need it to restart even after `docker stop`.

**Override files** — separate dev/prod config cleanly:
```bash
# dev: merges docker-compose.yml + docker-compose.override.yml automatically
docker compose up

# prod: explicit merge
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Networking

Docker networking determines how containers communicate. Choose the right driver:

| Driver | Use case |
|---|---|
| `bridge` | Default. Isolated network for services on same host. |
| `host` | Container shares host network stack. Max performance, no isolation. |
| `overlay` | Multi-host (Docker Swarm / custom). |
| `none` | No networking. Maximum isolation. |

**Service discovery:** On user-defined bridge networks, containers reach each other
by **service name** (the key in `services:`). DNS is automatic. `db:5432` just works.

Never use the default bridge network — it doesn't support DNS service discovery by name.

```yaml
# Good: explicit named network, use service name for discovery
networks:
  backend:

services:
  app:
    networks: [backend]
    environment:
      DB_HOST: db      # 'db' resolves to the db container's IP
  db:
    networks: [backend]
```

**Exposing vs publishing ports:**
- `expose: ["5432"]` — documents the port, makes it reachable to other containers only
- `ports: ["5432:5432"]` — publishes to the host (reachable from outside)

For internal services (databases, caches), use `expose` — don't publish to host.

---

## Volume Management

| Type | Syntax | Use case |
|---|---|---|
| Named volume | `mydata:/data` | Persistent data, DB files |
| Bind mount | `./src:/app/src` | Dev: live code reload |
| tmpfs | `type: tmpfs, target: /tmp` | Fast ephemeral scratch space |
| Anonymous | `/data` (no name) | Don't use — hard to manage |

Named volumes are managed by Docker and survive container removal. Bind mounts
map host paths directly — great for development but require absolute paths in production.

```yaml
# tmpfs (in-memory, fast, disappears on stop)
services:
  app:
    tmpfs:
      - /tmp
      - /run
```

---

## Security Best Practices

### Run as Non-Root

Containers run as root by default — a container escape would have root on the host.
Always create and switch to a non-root user.

```dockerfile
# Create a system user with no login shell
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup --no-create-home appuser

# Set ownership before switching user
COPY --chown=appuser:appgroup . .

USER appuser
```

For distroless images, use the built-in nonroot variant: `gcr.io/distroless/base-noroot`

### Read-Only Filesystem

```yaml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp      # app needs to write somewhere? Use tmpfs
      - /run
```

### Drop Capabilities

Linux capabilities grant partial root privileges. Drop all, add back only what's needed.

```yaml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # only if binding to port < 1024
```

### Secrets Management

Never put secrets in `ENV`, `ARG`, or image layers — they're visible in `docker inspect`
and image history.

**Docker secrets (Swarm/Compose v3):**
```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt   # or use external: true for Swarm

services:
  db:
    secrets: [db_password]
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
```

**BuildKit secret mounts (build-time):**
```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm install
```

```bash
docker build --secret id=npm_token,src=$HOME/.npmrc .
```

This keeps secrets out of the image entirely — they're mounted at build time only.

---

## Health Checks & Graceful Shutdown

### Healthcheck

Define in Dockerfile or Compose. Used by orchestrators to route traffic only to
healthy containers.

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

For services without `curl`, use `wget` or a custom binary:
```dockerfile
HEALTHCHECK CMD wget -qO- http://localhost:8080/health || exit 1
```

### Graceful Shutdown

Docker sends `SIGTERM` when stopping a container. Your app must handle it.
Always use the **exec form** of `CMD`/`ENTRYPOINT` so signals reach your process
directly — not a shell wrapper.

```dockerfile
# BAD: shell form — SIGTERM goes to sh, not your app
CMD python app.py

# GOOD: exec form — SIGTERM goes directly to python
CMD ["python", "app.py"]

# GOOD: explicit entrypoint + cmd split
ENTRYPOINT ["python"]
CMD ["app.py"]
```

Give containers time to shut down gracefully:
```yaml
services:
  app:
    stop_grace_period: 30s   # default is 10s
```

---

## Debugging & Troubleshooting

```bash
# Interactive shell in running container
docker exec -it <container> sh       # alpine
docker exec -it <container> bash     # debian/ubuntu

# Shell in a new container from image (good for inspecting)
docker run --rm -it <image> sh

# Logs (follow, last 100 lines)
docker logs -f --tail=100 <container>

# Resource usage
docker stats <container>

# Full container config + state
docker inspect <container>

# What's taking up space?
docker system df
docker image history <image>   # see layer sizes

# Remove all stopped containers, unused images, networks, build cache
docker system prune -af --volumes   # ⚠️ destructive

# Copy files out of container
docker cp <container>:/path/to/file ./local/

# Diff since container started
docker diff <container>
```

**Debugging a crashed container** (exits immediately):
```bash
# Override entrypoint to get a shell instead
docker run --rm -it --entrypoint sh <image>
```

---

## Registry Operations

```bash
# Tag convention: registry/namespace/name:tag
docker tag myapp:latest registry.example.com/team/myapp:1.2.3
docker tag myapp:latest registry.example.com/team/myapp:latest

# Push
docker push registry.example.com/team/myapp:1.2.3
docker push registry.example.com/team/myapp:latest

# Pull
docker pull registry.example.com/team/myapp:1.2.3

# Login (credentials stored in ~/.docker/config.json)
docker login registry.example.com

# Multi-platform build and push (requires buildx)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registry.example.com/team/myapp:1.2.3 \
  --push .
```

**Tagging strategy:**
- Always push both a version tag AND `latest`
- In CI: use git SHA for immutable tags + semantic version for human-friendly tags
- Never rely solely on `latest` in production

---

## CI/CD Integration

### GitHub Actions

```yaml
name: Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha     # GitHub Actions cache
          cache-to: type=gha,mode=max
```

### GitLab CI

```yaml
build:
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_BUILDKIT: "1"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest
        --build-arg BUILDKIT_INLINE_CACHE=1
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

---

## Language-Specific Patterns

See `references/stacks.md` for complete, production-ready Dockerfile patterns for:
- **Node.js** — npm ci, non-root, multi-stage for TypeScript
- **Python** — pip, Poetry, uv, virtual env patterns
- **Go** — static binary, scratch/distroless final stage
- **Java** — Maven/Gradle multi-stage, JRE-only final stage

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Using `latest` base image | Pin to `FROM node:20.11-alpine3.19` |
| Secrets in ENV/ARG | Use BuildKit secret mounts or Docker secrets |
| No `.dockerignore` | Create one — `node_modules` alone can be 500MB+ |
| Shell form CMD | Use exec form `["python", "app.py"]` for signal handling |
| Running as root | Add `USER` instruction with non-root user |
| Huge image from build tools | Multi-stage build, discard builder stage |
| Layer with `apt-get update` alone | Always combine with install in one `RUN` |
| Anonymous volumes | Use named volumes for anything you care about |
| `depends_on` without healthcheck | Add healthcheck + `condition: service_healthy` |
