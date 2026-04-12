# PRD: hermes-firman MVP -- Hermes Container Template

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).

## What hermes-firman Is

The default container template (firman) for Sultanate. Defines how to run a
Hermes-based province using the upstream Hermes Agent Docker image with
Sultanate-specific configuration.

A firman is the office. Who works there (personality, tools, permissions) is
defined by the berat.

## Key Insight: No Custom Dockerfile

Hermes Agent ships an official Docker image (v0.6.0+) that already includes:

- Debian 13.4 base with Python, Node.js, ripgrep, Playwright/Chromium
- Non-root `hermes` user with gosu privilege dropping
- Entrypoint that bootstraps `.env`, `config.yaml`, `SOUL.md` on first run
- Volume mount at `/opt/data` for all persistent state (sessions, memories,
  skills, workspace, etc.)
- Gateway mode for Telegram (polling + webhook)
- Profiles system for isolated instances

hermes-firman does NOT build a custom image. It uses the upstream
`nousresearch/hermes-agent` image and configures it through volume mounts
and environment variables at container creation time.

## What hermes-firman Defines

The delta between a stock Hermes container and a Sultanate province:

1. **Proxy configuration** -- `HTTP_PROXY`/`HTTPS_PROXY` env vars pointing
   to Janissary, plus Sultanate CA cert for HTTPS MITM
2. **Workspace bootstrap** -- clone target repo into `/opt/data/workspace`
3. **Berat application** -- write berat's SOUL.md, AGENTS.md, config.yaml
   into the volume, overriding Hermes defaults
4. **Telegram bot token** -- injected into config (low-risk secret, handled
   by Vizier)
5. **Janissary MCP server** -- added to config.yaml's `mcp_servers` section

## Province Bootstrap Sequence

Vizier executes these steps when creating a province:

```
1. docker create from nousresearch/hermes-agent image
   -> internal Docker network only (no external route)
   -> env: HTTP_PROXY, HTTPS_PROXY -> Janissary
   -> env: HERMES_HOME=/opt/data
   -> volume: /opt/data (province-specific, persistent)

2. docker cp: Sultanate CA cert into container's trust store

3. docker start: entrypoint runs, creates default dirs

4. docker exec: clone target repo into /opt/data/workspace
   (via Janissary proxy, credentials injected by Janissary)

5. docker exec: apply berat
   -> overwrite /opt/data/SOUL.md (from berat soul template)
   -> write /opt/data/workspace/AGENTS.md (from berat instructions)
   -> overwrite /opt/data/config.yaml (from berat tool/model/MCP config)

6. docker exec: hermes gateway (or restart container with gateway args)
   -> Hermes loads SOUL.md, config.yaml, AGENTS.md
   -> connects to Telegram via bot token in config
   -> Pasha is now reachable by Sultan
```

## Province Parameters (Firman-level)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `repo` | yes | -- | GitHub repo to clone |
| `branch` | no | `main` | Branch to check out |
| `name` | no | auto-generated | Province display name |

## Image Versioning

hermes-firman pins a specific Hermes Agent image tag for stability. Vizier
pulls this tag when creating provinces. Sultan can override the tag per
province if needed.

## Phase 1 Scope

**In scope:**
- Province creation using upstream Hermes Agent Docker image
- Proxy env vars + CA cert injection
- Workspace bootstrap (repo clone)
- Berat application (SOUL.md, AGENTS.md, config.yaml overwrite)
- Telegram bot token injection
- Janissary MCP server configuration

**Deferred:**
- Custom Dockerfile / image layers on top of upstream
- Multiple firman variants (openhands-firman, crewai-firman)
- Multi-stage bootstrap (clone + additional setup scripts)
- Post-task cleanup and artifact collection
