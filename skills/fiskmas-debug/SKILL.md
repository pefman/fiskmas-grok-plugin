---
name: fiskmas-debug
description: Debug a Fiskmås app that is unhealthy. Use when an app returns 502, is in a crash loop, fails its health check, or when deciding between restart and redeploy.
---

# Fiskmås debugging

Start with `get_status` (status, url, digest, last error), then `get_logs`, and
`get_stats` when you suspect resource pressure.

## 502 Bad Gateway

The proxy reached the host but the app was not serving.
- `get_status` → if `stopped`, the workload died; `start_project` or redeploy.
- `get_logs` → check the app bound to `0.0.0.0:$PORT` (not `127.0.0.1`)
  and that `$PORT` is `8080`. A server listening on the wrong address fails
  silently behind the proxy.
- Restart the workload, then redeploy if the binary/config is wrong.

## Crash loop (status flips running → failed)

- `get_logs` → look for the fatal line (missing env, bad port, OOM).
- OOM? The app exceeded the memory cap (256 MB Free / 512 MB Plus). Reduce usage or fix
  the leak.
- Missing env? `set_env` the required vars, then `deploy_project` on
  production (a restart is only a scale-up on the k8s backend and does not
  re-apply env; in local dev a restart does).

## Health check fails

- Confirm the health path exists (default `/health` or `/`).
- Confirm it responds on `127.0.0.1:$PORT` *inside* the app.
- A deploy is only cut over when healthy; on failure the previous workload is
  kept (if any) and the deploy is marked `failed`.

## Resource pressure (memory / CPU)

`get_stats` shows live CPU and memory against the plan caps (256 MB / 0.2 CPU free, 512 MB / 1 CPU plus)
and deploy history. Pass a `project` for a single project's view.

- Memory near the plan cap (256 MB free / 512 MB plus) → OOM risk; the container gets killed and the
  project is marked `failed`. Reduce usage (caches, batch sizes) or fix the
  leak, then `restart_project`.
- CPU pinned at the plan CPU cap (0.2 CPU free / 1 CPU plus) → the app is throttled; optimize hot paths.

## restart vs redeploy

- **restart** — cheap; restarts the same image. Use for transient crashes. On
  the production (k8s) backend a restart is only a scale-up and does NOT
  re-apply env — after any `set_env`, redeploy instead. `restart_project` is
  NOT rate-limited (deploys are: 1 per 5 minutes per account), so when a
  deploy fails with a `deploy rate limit` error, prefer it while you wait.
- **redeploy** — pulls the new digest (or re-applies the current one) and runs
  the workload with the stored env, health-gated. Use after pushing a new image
  AND after `set_env` on production.

## Database unreachable (ENOTFOUND on the MariaDB host)

If logs show `ENOTFOUND` for the MariaDB host (e.g. `shared-mariadb`) even
though `get_status` says running/healthy, the workload is older than the
platform's current DB hostname: the app's readiness probe is its own `/health`
and does not check the DB. The stored env now carries the correct
hostname (the platform migrates it on boot), so `deploy_project` to re-create
the workload with the current env. Do not use `restart_project` for this on
production — it keeps the old container env. Never `set_env` a custom
`DATABASE_URL`/`MYSQL_*` over the injected ones; that replaces the whole env
map and drops the provisioned credentials.

## URL shows a Fiskmås placeholder page

Not an error: before the first `deploy_project`, `<slug>-app.fiskmas.dev`
serves a small status-aware placeholder from the control plane, so the host
never 404s. The page states the project status. To make it serve the app,
push the image and `deploy_project`; the URL flips to the app once the deploy
is healthy. Only debug further if the page shows `failed`/`stopped` or the app
still placeholders after a healthy deploy (then check the per-deploy routing
with `get_status` → url).

## Escalate (feedback loop)

If you exhaust the above and still can't explain it, and you believe the
problem is the platform (not your app) — or the docs/skills were unclear and
led you astray — call `send_feedback` with `kind=bug` (or `kind=suggestion`
for unclear docs) and name the `tool` and `project` in the message. Do not
use it for anything but problems.

## Quick checklist

1. `get_status` → read status + last error.
2. `get_logs --tail 200` → find the fatal line.
3. `get_stats` → is it resource pressure (mem near the plan cap, CPU at the plan cap)?
4. Fix port (`0.0.0.0:8080`), env, memory, or queries.
5. `restart_project` for crashes; `deploy_project` for env changes or after a
   new push (production restarts do not re-apply env).
