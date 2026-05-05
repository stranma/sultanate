# PRD: openclaw-firman MVP -- OpenClaw Container Template

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For detailed schema and resolution see [OPENCLAW_FIRMAN_SPEC.md](OPENCLAW_FIRMAN_SPEC.md).

## What openclaw-firman Is

The default container template (firman) for Sultanate. Defines how to run
an OpenClaw-based province using the upstream OpenClaw Docker image with
Sultanate-specific configuration.

A firman is the office. Who works there (personality, tools, permissions)
is defined by the berat.

## Key Insight: No Custom Dockerfile

OpenClaw ships an official Docker image (`openclaw/openclaw:vYYYY.M.D`)
that already includes:

- A Debian-based runtime with Python, Node.js, ripgrep, and the
  `openclaw` CLI
- The OpenClaw gateway daemon, entrypoint, and auto-load logic for
  workspace files (`SOUL.md`, `AGENTS.md`, `IDENTITY.md`, etc.)
- Support for configuring via `~/.openclaw/openclaw.json` (or an
  explicit path via `OPENCLAW_HOME`)
- Multi-provider model support (Anthropic / OpenAI / OpenRouter /
  Ollama / LiteLLM) via a single config field

openclaw-firman does NOT build a custom image. It uses the upstream
`openclaw/openclaw` image and configures it through volume mounts and
environment variables at container creation time.

## What openclaw-firman Defines

The delta between a stock OpenClaw container and a Sultanate province:

1. **Network routing** -- WireGuard transparent proxy routes all traffic
   through Janissary, plus Sultanate CA cert for HTTPS MITM
2. **Workspace bootstrap** -- clone target repo into
   `/opt/data/workspace`
3. **Berat application** -- write berat's `SOUL.md`, `AGENTS.md`,
   `IDENTITY.md`, and `.openclaw/openclaw.json` into the volume
4. **Telegram bot token** -- injected into `openclaw.json`
   `channels.telegram` block (low-risk secret, handled by Vizier)
5. **Janissary HTTP API** -- appeal / request_access endpoints
   registered as an MCP server in `openclaw.json`
6. **OpenClaw sandbox mode** -- explicitly set to `"off"` / `"main"`;
   Sultanate's outer container already isolates the Pasha, so
   nesting OpenClaw's own Docker-in-Docker sandbox is unnecessary
   for MVP.

## Province Bootstrap Sequence

Vizier executes these steps when creating a province. See
[VIZIER_SPEC.md](VIZIER_SPEC.md) §5 for the full code.

```
1. docker create from openclaw/openclaw:vYYYY.M.D image
   -> internal network only (no external route)
   -> traffic routed through Janissary via WireGuard transparent proxy
   -> env: OPENCLAW_HOME=/opt/data, JANISSARY_API=http://10.13.13.1:8081
   -> volume: /opt/data (province-specific, persistent)

2. docker cp: Sultanate CA cert into container's trust store
   -> docker exec: update-ca-certificates

3. docker start: container comes up with the volume mounted

4. docker exec: clone target repo into /opt/data/workspace
   (via Janissary proxy, GitHub App token injected by Janissary)

5. docker exec mkdir -p /opt/data/.openclaw /opt/data/workspace

6. docker exec tee (via stdin pipe) writes the berat files:
   -> /opt/data/workspace/SOUL.md
   -> /opt/data/workspace/AGENTS.md
   -> /opt/data/workspace/IDENTITY.md       (optional)
   -> /opt/data/.openclaw/openclaw.json

7. docker exec -d: OPENCLAW_HOME=/opt/data openclaw gateway --port 18789
   -> OpenClaw loads openclaw.json from ~/.openclaw (which is
      /opt/data/.openclaw given OPENCLAW_HOME)
   -> OpenClaw scans the workspace and auto-loads SOUL.md / AGENTS.md /
      IDENTITY.md into context at first session turn
   -> connects to Telegram via the bot token in openclaw.json
   -> Pasha is now reachable by Sultan
```

## Province Parameters (Firman-level)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `repo` | yes | -- | GitHub repo to clone (owner/name) |
| `branch` | no | `main` | Branch to check out |
| `name` | no | auto-generated | Province display name |

## Image Versioning

openclaw-firman pins a specific OpenClaw image tag for stability. Deploy
scripts pin by digest (SHA256) to ensure reproducibility. Vizier pulls
this tag when creating provinces. Sultan can override the tag per
province at `create` time if needed.

## Phase 1 Scope

**In scope:**
- Province creation using upstream OpenClaw Docker image
- WireGuard proxy routing + CA cert injection
- Workspace bootstrap (repo clone)
- Berat application (SOUL.md, AGENTS.md, IDENTITY.md, openclaw.json)
- Telegram bot token injection via berat template
- Janissary security MCP server configuration
- OpenClaw sandbox mode: off (Sultanate's outer container is the
  isolation boundary)

**Deferred:**
- Custom Dockerfile / image layers on top of upstream
- Multiple firman variants (openhands-firman, crewai-firman) for
  non-OpenClaw runtimes
- Multi-stage bootstrap (clone + additional setup scripts)
- Nested OpenClaw sandboxing (Docker-in-Docker or SSH backend) for
  defense-in-depth against a compromised Pasha subsession
- Post-task cleanup and artifact collection
