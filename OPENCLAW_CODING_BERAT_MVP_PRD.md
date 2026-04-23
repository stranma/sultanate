# PRD: openclaw-coding-berat MVP -- Coding Agent Profile

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For the container template see [OPENCLAW_FIRMAN_MVP_PRD.md](OPENCLAW_FIRMAN_MVP_PRD.md).
> For detailed schema and rendering see [OPENCLAW_CODING_BERAT_SPEC.md](OPENCLAW_CODING_BERAT_SPEC.md).

## What openclaw-coding-berat Is

The default agent profile (berat) for Sultanate. Defines who the Pasha is:
personality, operating rules, tools, and security policy for an OpenClaw
agent doing software development work.

A berat is the employee. Where they work is defined by the firman.

## What It Defines

1. **Soul** -- Pasha personality and operating style (`SOUL.md`)
2. **Instructions** -- operating rules and role definition (`AGENTS.md`)
3. **Identity** (optional) -- agent identity block (`IDENTITY.md`)
4. **OpenClaw configuration** (`openclaw.json`) -- model, tools, MCP
   servers, channels
5. **Security policy** -- whitelist, grants, non-HTTP port declarations

## Templating

Vizier fills `{{variable}}` placeholders at province creation time.

| Variable | Source | Example |
|----------|--------|---------|
| `{{province_id}}` | Auto-generated | `prov-a1b2c3` |
| `{{province_name}}` | Sultan or auto-generated | `backend-refactor` |
| `{{pasha_name}}` | Sultan or berat default | `Kemal` |
| `{{repo_name}}` | Sultan (required) | `stranma/EFM` |
| `{{extra_instructions}}` | Sultan (optional) | `Prefer small commits...` |
| `{{model}}` | Sultan or berat default | `anthropic/claude-sonnet-4` |
| `{{pasha_telegram_bot_token}}` | Vizier (bot pool) | `123456:ABC-def...` |
| `{{sultan_telegram_user_id}}` | Deploy-time env var | `123456789` |
| `{{janissary_api}}` | Vizier-injected | `http://10.13.13.1:8081` |

Missing optional variables are replaced with empty string.

## Soul

Written to `/opt/data/workspace/SOUL.md` by Vizier. OpenClaw auto-loads
this at the first session turn.

```markdown
You are {{pasha_name}}, a Pasha in the Sultanate system. You work in
province {{province_name}} on repository {{repo_name}}.

You execute tasks assigned by Sultan with precision and transparency.
You report progress honestly. If you're stuck, you say so. If you need
access to something outside your whitelist, you request it through the
Janissary security MCP tool -- never by trying to work around the
proxy. If a request is blocked and the block seems wrong, appeal it
with a clear justification. Be direct. Be honest.
```

## Instructions

Written to `/opt/data/workspace/AGENTS.md` by Vizier. OpenClaw auto-loads
this as the project-specific context.

```markdown
# Working Rules

- Work only within /opt/data/workspace (the repo clone)
- Use the janissary_security MCP tool to appeal blocked requests or
  to request new access (credentials, extra domains, non-HTTP ports)
- Commit to a feature branch, not main
- Create a pull request when your task is complete
- Do not attempt to read or guess secret values -- you should never
  see credentials. If a request works, it works; if it returns 401/403,
  appeal or ask for access

{{extra_instructions}}
```

## Identity (optional)

Written to `/opt/data/workspace/IDENTITY.md`. Short per-agent identity
line. Can be omitted.

```markdown
{{pasha_name}} 👷
```

## Tools

### OpenClaw Built-in Tools

Curated subset enabled by default. OpenClaw has more; the berat picks a
conservative set:

| Tool | Purpose |
|------|---------|
| `bash` | Shell commands in the workspace (primary) |
| `read` | File read |
| `write` | File write |
| `edit` | Line-oriented file editing |
| `browser` | Browser automation (headless Chromium; optional) |
| `canvas` | Scratchpad for iterative drafting |
| `nodes` | Task-graph reasoning (optional in MVP) |

Tools excluded by default: `process` (long-running subprocesses),
`cron` (scheduled jobs), `discord` (no Discord integration), `gateway`
(meta-tool, not for agent use), `sessions_*` (session introspection).
Sultan can enable these per-province at `create` time.

### Janissary Security MCP

Two tools provided by Janissary's HTTP API, registered as an MCP server
in `openclaw.json`:

**`appeal_request`** -- appeal a blocked outbound request:

```
appeal_request(url, method, payload, justification)
```

Janissary forwards the full payload + justification to Kashif for
triage. Verdict is written to Divan; Janissary picks it up on its next
poll and applies on retry. See `ARCHITECTURE.md` appeal flow for the
full timeline.

**`request_access`** -- request new credentials or access (e.g., a new
API, a new domain for the whitelist):

```
request_access(service, scope, justification)
```

Text is Kashif-screened, then relayed by Vizier to Sultan. Sultan tells
Aga to provision.

### OpenClaw Configuration

Written to `/opt/data/.openclaw/openclaw.json` by Vizier:

```json
{
  "agent": {
    "model":     "{{model}}",
    "workspace": "/opt/data/workspace"
  },
  "agents": {
    "defaults": {
      "workspace": "/opt/data/workspace",
      "sandbox":   { "mode": "off" }
    }
  },
  "tools": {
    "exec": { "applyPatch": false }
  },
  "channels": {
    "telegram": {
      "botToken":  "{{pasha_telegram_bot_token}}",
      "allowFrom": [ "{{sultan_telegram_user_id}}" ],
      "dmPolicy":  "pairing"
    }
  },
  "mcp_servers": {
    "janissary_security": {
      "transport": "http",
      "url":       "{{janissary_api}}/mcp"
    }
  }
}
```

Sultan can override `model` at province creation time. The berat
default is `anthropic/claude-sonnet-4`; provider format is
`"provider/model-id"` (Anthropic / OpenAI / OpenRouter / Ollama / LiteLLM).

`sandbox.mode = "off"` because the Pasha is already isolated by
Sultanate's outer container (WireGuard tunnel + iptables kill-switch).
Nesting OpenClaw's own Docker-in-Docker sandbox is not needed for MVP.

## Security Policy

Vizier writes these defaults to Divan when creating the province. Aga
reads them and provisions grants accordingly.

### Default Whitelist

Domains the province can access without restriction (see
`DIVAN_API_SPEC.md` for schema):

| Domain | Reason |
|--------|--------|
| `github.com` | Repo operations (clone, push, PR) |
| `api.github.com` | GitHub API |
| `pypi.org` | Python packages |
| `files.pythonhosted.org` | Python package downloads |
| `registry.npmjs.org` | Node packages |
| `cdn.jsdelivr.net` | CDN for npm packages |
| `docs.python.org` | Python documentation |
| `stackoverflow.com` | Developer reference |

Sultan can expand or restrict per province via Aga.

### Default Grants

| Grant | Domain | Injection | Provisioned By |
|-------|--------|-----------|----------------|
| GitHub repo access | `api.github.com`, `github.com` | `Authorization: Bearer <GitHub-App-token>` | Aga mints via GitHub App (dynamic mode, 1-hour TTL, auto-renewed) |

The Pasha never sees the token. Janissary injects at the proxy level.
Aga handles minting, renewal, and revocation -- Sultan does not paste
tokens (see `SULTANATE_MVP.md` GitHub Token Strategy).

Additional grants require Sultan approval via Aga.

### Non-HTTP Port Declarations

| Host | Port | Protocol | Reason |
|------|------|----------|--------|
| `github.com` | 22 | TCP | Git SSH (optional; HTTPS is default) |

All port openings require Sultan approval. Vizier writes declarations
to Divan, Aga asks Sultan, then opens on approval via iptables.

## Province Parameters (Berat-level)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `pasha_name` | no | `Pasha` | Agent's name in SOUL.md |
| `extra_instructions` | no | empty | Additional instructions appended to AGENTS.md |
| `model` | no | berat default | LLM model override (e.g. `openai/gpt-5`) |
| `extra_whitelist` | no | `[]` | Additional whitelisted domains |
| `extra_tools` | no | `[]` | Additional OpenClaw built-in tools to enable |

## Phase 1 Scope

**In scope:**
- SOUL.md template for coding Pasha
- AGENTS.md template with variable substitution
- Optional IDENTITY.md template
- openclaw.json template (model, sandbox off, Telegram channel,
  Janissary security MCP)
- Curated OpenClaw built-in tool selection (bash, read, write, edit,
  browser, canvas, nodes)
- Janissary security MCP server configuration (HTTP transport)
- Default whitelist (GitHub, PyPI, npm, docs)
- Default GitHub grant (GitHub App, dynamic mode, 1-hour TTL)
- Non-HTTP port declarations (Sultan approval required)
- Berat-level province parameters (pasha_name, extra_instructions,
  model, extra_whitelist, extra_tools)

**Deferred:**
- Additional berats (research-berat, assistant-berat)
- Berat inheritance (base berat + overlays)
- Berat versioning and migration
- Enabling nested OpenClaw sandbox (Docker-in-Docker) for
  defense-in-depth
- Additional Telegram channels (e.g., a second bot for Sultan to
  broadcast to the whole fleet)
