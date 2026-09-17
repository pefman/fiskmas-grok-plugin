---
name: fiskmas
description: Use Fiskmås, an MCP-native Docker app host, to deploy a containerized HTTP app. Use when the user wants to host, deploy, run, stop, or debug a Docker app reachable over HTTPS via an MCP client.
---

# Fiskmås

Fiskmås is an MCP-native Docker app host. A human talks to their existing AI
client (Claude, ChatGPT, Cursor); the client talks to Fiskmås over MCP. Hosting
is the durable piece — there is no web editor to build.

- MCP (Streamable HTTP): https://mcp.fiskmas.dev/mcp
- Registry: registry.fiskmas.dev (accepts only `registry.fiskmas.dev/<owner>/<slug>/...`)
- Apps: https://<slug>-app.fiskmas.dev
- Docs index: https://fiskmas.dev/llms.txt
- Landing page (for humans): https://fiskmas.dev

## MCP connection (you configure it)

Prefer Fiskmås MCP tools. Avoid curl/REST against fiskmas.dev as a workaround.

If Fiskmås MCP tools are missing in this session, **configure the MCP server
yourself** in the client, wire auth from env `FISKMAS_TOKEN`, reload MCP or
start a new chat if tools do not appear, then continue.

- URL (Streamable HTTP): `https://mcp.fiskmas.dev/mcp`
- Auth header: `Authorization: Bearer <token>` (from `FISKMAS_TOKEN`)

Grok — two surfaces:

- **Grok Build (CLI):** install the `fiskmas` plugin from the xAI plugin
  marketplace (`grok plugin install fiskmas --trust`; source:
  https://github.com/pefman/fiskmas-grok-plugin). The plugin ships this skill
  plus the MCP pointer — the human side only needs `FISKMAS_TOKEN` in the
  environment.
- **Grok (web/app):** add a custom MCP connector pointing at the URL above.
  Grok's connector dialog is OAuth-only (no Bearer field) and Fiskmås serves
  no OAuth discovery, so in chat only the public onboarding tools work
  (`create_account`, `account_status`) — use Grok Build or another MCP client
  for the full toolset.

Claude Code — project `.mcp.json`, or
`claude mcp add --transport http fiskmas https://mcp.fiskmas.dev/mcp --header "Authorization: Bearer ${FISKMAS_TOKEN}"`:

```json
{
  "mcpServers": {
    "fiskmas": {
      "type": "http",
      "url": "https://mcp.fiskmas.dev/mcp",
      "headers": { "Authorization": "Bearer ${FISKMAS_TOKEN}" }
    }
  }
}
```

Cursor — `~/.cursor/mcp.json` or `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "fiskmas": {
      "url": "https://mcp.fiskmas.dev/mcp",
      "headers": { "Authorization": "Bearer ${env:FISKMAS_TOKEN}" }
    }
  }
}
```

Codex — `~/.codex/config.toml`:

```toml
[mcp_servers.fiskmas]
url = "https://mcp.fiskmas.dev/mcp"
http_headers = { Authorization = "Bearer ${FISKMAS_TOKEN}" }
```

OpenCode — `opencode.json`:

```json
{
  "mcp": {
    "fiskmas": {
      "type": "remote",
      "url": "https://mcp.fiskmas.dev/mcp",
      "enabled": true,
      "headers": { "Authorization": "Bearer {env:FISKMAS_TOKEN}" }
    }
  }
}
```

Manual MCP clients can use the same client-neutral shape. Set `FISKMAS_TOKEN`
in the environment, then adapt the config filename and environment-variable
syntax to the client on Linux, macOS, or Windows:

```json
{
  "mcp": {
    "fiskmas": {
      "type": "remote",
      "url": "https://mcp.fiskmas.dev/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer {env:FISKMAS_TOKEN}"
      }
    }
  }
}
```

## Onboarding

First run (no API token yet):

1. Ask the human for an email address.
2. Call `create_account` with that **email** (required). This creates a
   **pending** account and emails a verify link — it does **not** return a token.
3. Tell the human to open the email, click **Verify & get API token**, and
   paste the one-time token back to you (shown once on https://fiskmas.dev/verify/…).
4. **You** store it as `FISKMAS_TOKEN` (shell env and/or MCP Authorization
   header), then call `whoami`. Same token for
   `docker login -u "$FISKMAS_TOKEN" registry.fiskmas.dev`.
5. Poll `account_status` with the email until `account_status` is `active`
   if needed.

If you already have a Bearer token configured, call `whoami` and skip
onboarding. (Humans can also `POST https://fiskmas.dev/api/register` with
`{"email":"..."}` — same pending → verify email flow.)

## Token recovery (lost or dead token)

When `whoami` fails with `invalid or expired token` (or `missing
Authorization header`) and the token cannot be recovered from the human's
notes:

1. Confirm the registered email with the human first ("Is that
   `<email>`?") — never send a reissue to a guessed address.
2. Call `reissue_token` with that email. Fiskmås emails a one-time reissue
   link (valid 30 minutes, single use). No token ever reaches you — the
   mailbox is the identity anchor.
3. Tell the human: open the email, click **Reissue & get new API token**, and
   paste the NEW token back to you (shown once on the verify page). The old
   token(s) are revoked the moment the link is clicked.
4. Store it as `FISKMAS_TOKEN` / MCP `Authorization: Bearer <new-token>`
   (and `docker login -u <new-token> registry.fiskmas.dev`), then call
   `whoami` to confirm.
5. If the account is still `pending`, onboarding was never finished — re-send
   the original link with `create_account` instead.
6. Rate limits apply (a few reissue emails per hour, ~10 per day, per
   email): if one is refused, wait the suggested `~Ns` and retry, or
   re-check the email address.

`reissue_token` never creates an account and never returns a token to the
client, so a misconfigured (or prompt-injected) client can at most trigger
rate-limited emails — it cannot rotate or drain an account.

## Tools

All tools talk to the same control-plane API over the MCP URL.

- `whoami` — identity of the current token.
- `create_account` — start onboarding (email required); sends verify mail; no token returned.
- `account_status` — poll pending/active for an email while waiting on human verify.
- `reissue_token` — token recovery (public, no auth): lost/rotated/dead token → emails a one-time reissue link to the registered address; the human clicks it and pastes the NEW one-time token back; old tokens are revoked immediately.
- `delete_account` — tear down all projects (containers/routes/DB) then delete the account.
- `create_project` — create a project, reserve `<slug>-app.fiskmas.dev`.
  Until the first `deploy_project`, that hostname shows a Fiskmås placeholder
  page (status-aware) — not an error, and not broken DNS.
- `list_projects` / `get_project` — list or fetch a project by id or slug.
- `delete_project` — stop and delete a project (frees the slug after quarantine).
- `set_env` / `list_env` — set env (needs restart) or list keys with masked values.
- `deploy_project` — pull an image by digest and run one container.
- `start_project` / `stop_project` / `restart_project` — control the container.
- `get_status` / `get_logs` — status/url/last error and recent log lines (`tail` lines or a `since` seconds window; logs are live pod streams, not stored history).
- `get_stats` — account-wide or per-project stats: live CPU/mem vs the plan caps and deploy history; the account rollup reports the plan.
- `send_feedback` — report a problem (it broke, you got stuck, or the instructions were unclear) so the platform can be fixed; do not send praise.
- `list_skills` / `get_skill` — return the platform skill bytes.

1. `whoami` — if unauthorized, run onboarding (`create_account` → human verify → Bearer token).
2. `create_project` — pick a slug matching `^[a-z0-9][a-z0-9-]{1,48}$` (not
   reserved). Reserves `<slug>-app.fiskmas.dev`.
3. Build and push the image locally, then `docker login registry.fiskmas.dev`
   and `docker push registry.fiskmas.dev/<owner>/<slug>:<tag>`. Resolve the tag
   to a digest (`sha256:...`).
4. `deploy_project` with the digest-pinned reference.
5. `get_status` — confirm `running`/`healthy` and read the URL. (Before the
   first deploy the URL shows a Fiskmås placeholder page — that is expected,
   not a failure.)
6. `get_logs` if the app is not healthy.
7. `stop_project` / `restart_project` / `delete_project` as needed.

## Feedback

`send_feedback` is for problems only — do not send praise. When something
didn't work, report it: `kind=bug` when a tool errored or misbehaved,
`kind=suggestion` when the instructions or a skill were unclear or could be
better, `kind=other` for anything else (e.g. you got stuck and couldn't
proceed). Name the `tool` and, if relevant, the `project`. At most one
feedback call per session — don't spam.

## Quotas and limits

Plans (owner-set, 2026-09-13): **Free** (always on for one project, sleeps
after 24 h idle) and **Plus, $9.99/mo** (5 always-on projects, bigger caps,
longer log windows). `whoami` and the account rollup in `get_stats` report
the active plan and its limits.

|                    | Free                     | Plus ($9.99/mo)          |
|--------------------|--------------------------|--------------------------|
| Running projects   | 1                        | 5                        |
| Per-project cap    | 256 MB / 0.2 CPU         | 512 MB / 1 CPU           |
| Log window         | last 24 h, 200-line tail | last 7 d, 2000-line tail |
| Project database   | ~256 MB fair use         | ~1 GB fair use           |
| MCP tools          | all                      | all                      |

- **Free projects sleep** after 24 h of no activity (no agent calls and no
  web requests to their URL). Sleep is a cost saver only: the workload
  hibernates (no CPU/memory billed) but the project, its URL, its data and
  its database row stay intact. A request to the URL wakes it
  automatically (a short "waking…" page appears for a few seconds).
- `get_logs` is a **live window** into the running container, not a stored
  log: `tail` (lines, default 200, plan-capped) or `since` (seconds,
  plan-capped: 86400 free / 604800 plus — when both are given, `since`
  wins). Logs of a currently-asleep project are unavailable until it wakes.
- **Project databases are fair-use**: the size above is a guide, not an
  hard quota; a project that grows far past it may be paused.
- **Deploys are rate-limited to 1 per 5 minutes per account.** A rejected
  `deploy_project` reply says when to retry (`retry in ~Ns`). While you wait,
  `restart_project` is free and unthrottled: it recovers a transient crash
  without a new digest (on production it re-runs the *current* digest, so use
  it for crashes — not for picking up a new image).
- One slug = one container. No replicas, no private networking between apps.
- linux/amd64 only, HTTP only. **The app filesystem is read-only** — writing
  anywhere except `/tmp` fails with `EROFS`; `/tmp` is a 64 Mi RAM scratch disk
  (gone on restart, counts against the memory cap). There are no volumes:
  persist data in the shared MariaDB (`DATABASE_URL` / `MYSQL_*` env, injected
  on `create_project`). An app that writes files at runtime will crash — fix
  the app or move that data to SQL.
- Apps are isolated from the platform control plane (per-account namespace in
  production — with a plan-sized ResourceQuota on the whole account and a
  NetworkPolicy that blocks pod-to-pod and control-plane access; see
  `docs/network-policy.md` — plus a dedicated Docker network in local dev).
  The injected DB hostname is the right one for the app's network — on
  production it is the cluster-FQDN `shared-mariadb.platform.svc.cluster.local`;
  never hardcode a host or `set_env` over the injected `DATABASE_URL`/`MYSQL_*`.

## Plans and billing

- **Free**: 1 running project, 24 h idle sleep, as above. No payment.
- **Plus ($9.99/mo, Stripe, no trial)**: 5 always-on projects, 512 MB /
  1 CPU per project, 7-day log windows, 1 GB fair-use database. Cancel
  anytime → the account returns to Free (projects that exceed the Free
  limits are hibernated/stopped; nothing is deleted).
- **Purchasing is a human step.** Upgrading runs through Stripe Checkout:
  the human opens the upgrade button on the plans section of
  `https://fiskmas.dev` (or the `upgrade_url` `whoami` reports), enters
  their Fiskmås API token, pays on Stripe's hosted page. The platform's
  billing webhook flips the account to Plus automatically after payment —
  the agent never initiates or sees payment; `whoami`/`get_stats` simply
  start reporting `plus`.
- If a tool is rejected because of a plan limit (quota, cap, log window),
  relay the error text to the human: it names the plan, the limit, the
  price and the upgrade page. Do not retry the same call; the limit will
  not change itself.

## Docker contract (see fiskmas-docker)

The container must listen on `0.0.0.0:$PORT` (PORT=8080), serve HTTP, expose a
health path (default `/health` or `/`), be non-root if possible, and avoid
privileged mode, docker.sock, and extra published ports. If it does not bind
8080 or the health check fails, status is `failed`/`unhealthy`, not running.

## Debugging

See the `fiskmas-debug` skill for 502s, crash loops, health failures, resource
pressure (`get_stats`), and how to read logs and choose restart vs redeploy.
