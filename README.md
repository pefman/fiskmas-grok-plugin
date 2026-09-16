# fiskmas-grok-plugin

Grok Build plugin for [Fiskmås](https://fiskmas.dev) — an MCP-native Docker app host. Your
AI client (Grok) talks to Fiskmås over MCP; Fiskmås hosts your containerized HTTP app at
`https://<slug>-app.fiskmas.dev` and manages it end to end.

This repo ships for Grok Build: an MCP server pointer plus the Fiskmås agent skills.
There is no code installed and nothing runs locally — the plugin only wires Grok to the
hosted MCP server at `https://mcp.fiskmas.dev/mcp`.

## What it does

- **Onboard in chat** — `create_account` emails you a verify link; the one-time token you
  paste back becomes your Fiskmås API token, reused for `docker login`.
- **Deploy** — build any containerized HTTP app locally, push to `registry.fiskmas.dev`,
  and Fiskmås runs it behind TLS on a durable host, with a per-project shared MariaDB.
- **Operate** — start/stop/restart, live logs and stats, env management, plan and quota
  visibility, and guided debugging (502s, crash loops, health failures).

## Install

Once listed in the xAI plugin marketplace (the normal path):

```bash
grok plugin install fiskmas --trust
```

Before it is listed, Grok Build can register this repo as a marketplace source
directly (Marketplace → Add → Git URL: `https://github.com/pefman/fiskmas-grok-plugin`)
or install a local clone:

```bash
grok plugin install ./fiskmas-grok-plugin
```

Claude Code reads the same files (root `.mcp.json` + `.claude-plugin/plugin.json` +
`skills/`), so the plugin also works there.

## One token, three surfaces

Everything uses a single Fiskmås API token:

- MCP: `Authorization: Bearer <token>` against `https://mcp.fiskmas.dev/mcp` (the
  `headers` block in `.mcp.json` already wires the `FISKMAS_TOKEN` env var)
- Docker registry: `docker login -u "$FISKMAS_TOKEN" registry.fiskmas.dev`
- REST: `Authorization: Bearer <token>` against `https://fiskmas.dev/api/*` (rarely needed)

Get one for free at https://fiskmas.dev (no card required — 1 running project included).

## Network endpoints this plugin calls

| Endpoint | Why |
|---|---|
| `https://mcp.fiskmas.dev/mcp` | The MCP server itself (tools like `deploy_project`) |
| `registry.fiskmas.dev` | `docker push` / `docker login` only — driven by your local Docker CLI on your behalf, not by the plugin |
| `https://<slug>-app.fiskmas.dev` | Your app's public URL, after deployment |
| `https://fiskmas.dev` | Human-facing landing page (account creation, upgrade) |

## Repository policy

**Never commit real tokens, keys, or credentials to this repo** — it is public and pinned
by SHA from the xAI plugin marketplace. The only secret this plugin uses is the `FISKMAS_TOKEN`
environment variable at runtime. If a token is ever accidentally committed, rotate it at
https://fiskmas.dev and force-push; the marketplace will pick up the new SHA on its next
daily bump.

The skills in `skills/` are copies of the skill source in the Fiskmås platform repo; they
are re-synced whenever the platform's MCP surface changes (same day, via the marketplace's
SHA-bump flow).

## License

MIT. Fiskmås is operated by Fiskmås (fiskmas.dev); contact: noreply@fiskmas.dev.