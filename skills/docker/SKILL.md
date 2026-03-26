---
name: docker
description: "Analyzes and generates production-ready Dockerfiles, Docker Compose configurations, and container security hardening. Optimizes image size via multi-stage builds, layer caching, and minimal base images. Reviews Dockerfiles for security issues (root user, secrets in layers, missing .dockerignore) and applies fixes. Use when docker, dockerfile, container, docker-compose, docker build, containerize, multi-stage build, image optimization, container security, or devops is mentioned."
---

# Docker

## Principles

- Use the smallest possible base image for production (alpine, slim, distroless)
- Separate build and runtime with multi-stage builds
- Run one process per container
- Order layers from least to most frequently changed for cache efficiency
- Never run as root in production containers
- Never bake secrets into images — inject at runtime
- Always include a `.dockerignore` alongside every Dockerfile

## Workflow

### Creating a Production Dockerfile

1. **Choose a minimal base image.** Prefer `node:20-alpine`, `python:3.12-slim`, or `gcr.io/distroless/*` over full OS images. Pin the version tag (e.g., `node:20.10-alpine`), never use `:latest`.
2. **Set up layer caching.** Copy dependency manifests (`package.json`, `requirements.txt`, `go.mod`) first, then run the install step. Copy application source code afterward so dependency layers stay cached.
3. **Use a multi-stage build.** Build in one stage, copy only the runtime artifacts into a clean final stage.
4. **Create a non-root user.** Add a group and user, set file ownership with `--chown`, and switch with `USER` before the final `CMD`.
5. **Add a health check.** Use `HEALTHCHECK` so orchestrators can detect unhealthy containers.
6. **Create a `.dockerignore`.** Exclude `.git`, `node_modules`, `.env*`, test files, IDE configs, and secrets.
7. **Validate.** Check against `references/validations.md` rules: no secrets in ARG/ENV, no `:latest` tag, no `ADD` when `COPY` suffices, no `COPY . .` before dependency install.

### Optimizing an Existing Dockerfile

1. **Audit the base image.** Replace full images (`node:20`, `ubuntu:22.04`) with alpine or slim variants. Measure size reduction with `docker images`.
2. **Reorder COPY and RUN.** Move dependency installation above source code copy. Combine multiple `RUN apt-get` commands into one layer and clean up caches (`rm -rf /var/lib/apt/lists/*`).
3. **Add multi-stage if missing.** Split build tools from the runtime image. For Go, the final stage can be `scratch` (approximately 5 MB total).
4. **Check security.** Verify a `USER` instruction exists, no secrets appear in `ARG`/`ENV`, and a `.dockerignore` is present. Consult `references/sharp_edges.md` for common vulnerabilities.
5. **Re-run build and compare.** Confirm the optimized image builds, passes health checks, and is smaller.

### Setting Up Docker Compose for Development

1. **Define services.** Map each component (app, database, cache) to its own service.
2. **Mount source code as a volume** for hot reload; keep `node_modules` as an anonymous volume.
3. **Use `depends_on` with health checks** so the app waits for the database to be ready.
4. **Inject environment variables** via the `environment` key — never hard-code secrets in the YAML.

## Example: Production Multi-Stage Dockerfile (Node.js)

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS production
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

## Example: .dockerignore

```
node_modules
.git
.env
.env.*
dist
coverage
tests
__tests__
*.md
docs
.vscode
.idea
*.pem
*.key
secrets/
Dockerfile*
docker-compose*
```

## Reference System Usage

Ground all responses in the provided reference files, treating them as the source of truth for this domain:

- **For creation:** Consult **`references/patterns.md`** for multi-stage builds, layer caching, Docker Compose, security hardening, health checks, and `.dockerignore` templates.
- **For diagnosis:** Consult **`references/sharp_edges.md`** for critical failures — secrets in layers, root user defaults, cache busting, latest tag risks, missing `.dockerignore`, and multi-process anti-patterns.
- **For review:** Consult **`references/validations.md`** for regex-based rules to validate Dockerfiles and Compose files (root user, secrets in args, unpinned tags, missing cleanup, missing health checks).

If a user's request conflicts with guidance in these references, explain the recommended approach using the reference material.
