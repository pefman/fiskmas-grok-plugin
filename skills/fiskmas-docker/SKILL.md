---
name: fiskmas-docker
description: Write a Fiskmås-compatible Dockerfile. Use when building an image to deploy to Fiskmås — PORT, health path, amd64, size and memory caps, non-root, and no secrets baked in.
---

# Fiskmås Docker contract

Fiskmås runs one container per project. Force Docker: build the image locally
and push to the registry; Fiskmås does not remote-build from git in v1.

## Required

- **HTTP only.** Listen on `0.0.0.0:$PORT`. Fiskmås injects `PORT=8080`. Bind
  to `0.0.0.0` (not `127.0.0.1`) or the proxy cannot reach it.
- **Health path.** Serve a health endpoint, default `/health` (or `/`). If it
  does not respond, the deploy is marked `failed`/`unhealthy`.
- **linux/amd64 only.** Multi-arch is ignored in v1.
- **Ephemeral filesystem.** No persistent volumes on the app container. Use the
  shared MariaDB: `DATABASE_URL` (and `MYSQL_*`) are injected on `create_project`.
  The DB host is injected for you (a cluster-FQDN on production) — do not
  hardcode it, and do not override it via `set_env` (that replaces the whole
  env and drops the credentials).
- **Non-root if possible.** Run as an unprivileged user (`USER`).
- **No secrets baked in.** Pass secrets via env (`set_env`), never `ARG`/`ENV`
  in the image. Env is encrypted at rest on the server.

## Hard limits

- **No privileged mode, no `docker.sock`.**
- **No extra published ports.** Only the proxied 8080 is exposed.
- **Image size cap ~1 GB.** Keep base images small (`alpine`, `distroless`,
  `scratch` where practical).
- **Memory cap.** Plan-driven: **256 MB** (Free) / **512 MB** (Plus) per project. OOM → `failed`.
- **CPU cap.** **0.2 CPU** (Free) / **1 CPU** (Plus) per project.

## Example

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
ENV PORT=8080
EXPOSE 8080
USER node
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/health || exit 1
CMD ["node", "server.js"]
```

## Push to the registry

```bash
# Your Fiskmås API token (shown once on the email verify page) is also your
# registry credential:
docker login -u <your-API-token> registry.fiskmas.dev
docker build -t registry.fiskmas.dev/<owner>/<slug>:v1 .
docker push registry.fiskmas.dev/<owner>/<slug>:v1
# resolve to a digest for a stable deploy:
docker inspect --format='{{.Config.Digest}}' ...   # or use registry API
```

Only `registry.fiskmas.dev/<owner>/<slug>/...` images run. Arbitrary
`docker.io` images are rejected so apps cannot run on your IP.
