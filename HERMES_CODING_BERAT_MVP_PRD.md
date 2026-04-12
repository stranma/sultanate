# PRD: hermes-coding-berat MVP -- Coding Agent Profile

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).

## What hermes-coding-berat Is

The default agent profile (berat) for Sultanate. Defines who the Pasha is:
personality, operating rules, tools, and security policy for a Hermes agent
doing software development work.

A berat is the employee. Where they work is defined by the firman (see
[HERMES_FIRMAN_MVP_PRD.md](HERMES_FIRMAN_MVP_PRD.md)).

## What It Defines

1. **Soul** -- Pasha personality and operating style (`SOUL.md`)
2. **Instructions** -- operating rules and role definition (`AGENTS.md`)
3. **Tools** -- Hermes built-in tools and MCP servers
4. **Security policy** -- whitelist, grants, non-HTTP port declarations

## Templating

Vizier fills variables at province creation time:

| Variable | Source | Example |
|----------|--------|---------|
| `{{province_id}}` | Auto-generated | `prov-a1b2c3` |
| `{{province_name}}` | Sultan or auto-generated | `backend-refactor` |
| `{{pasha_name}}` | Sultan or berat default | `Kemal` |
| `{{repo_name}}` | Sultan (required) | `stranma/EFM` |
| `{{extra_instructions}}` | Sultan (optional) | `Prefer small commits...` |

Missing optional variables are replaced with empty string.

## Soul

Written to `/opt/data/SOUL.md` by Vizier:

```markdown
You are {{pasha_name}}, a Pasha in the Sultanate system. You work in
province {{province_name}}.

You execute tasks assigned by Sultan with precision and transparency.
You report progress honestly. If you're stuck, you say so. If you need
access to something, you request it through the security tool. You never
try to work around security restrictions.
```

## Instructions Template

Written to `/opt/data/workspace/AGENTS.md` by Vizier. Hermes auto-loads
this file as project-specific context.

```markdown
# Working Rules

- Work only within /opt/data/workspace
- Use the security MCP tool to appeal blocked requests or request new access
- Commit to a feature branch, not main
- Create a PR when your task is complete

{{extra_instructions}}
```

## Tools

### Hermes Built-in Tools

Curated subset of Hermes's 40+ tools:

| Tool | Purpose |
|------|---------|
| `terminal` | Shell commands in workspace |
| `file` | File read/write/search |
| `web` | Web search and fetch |
| `browser` | Browser automation |
| `code_execution` | Sandboxed code execution |
| `memory` | Persistent agent memory |
| `todo` | Task tracking |
| `delegation` | Sub-agent spawning |
| `clarify` | Ask Sultan for clarification |

Sultan can enable additional tools per province.

### Janissary Security MCP

Two tools provided by Janissary's HTTP API (exposed as MCP tools in provinces):

**`appeal_request`** -- appeal a blocked outbound request:
```
appeal_request(url, method, justification)
```
Stored in Divan, relayed by Vizier to Sultan.

**`request_access`** -- request new credentials or access:
```
request_access(service, scope, justification)
```
Relayed to Sultan via Vizier. Sultan tells Sentinel to provision.

### Hermes Configuration

Written to `/opt/data/config.yaml` by Vizier:

```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"
  provider: openrouter

terminal:
  backend: local
  cwd: "/opt/data/workspace"
  timeout: 120

tools:
  - web
  - terminal
  - file
  - browser
  - code_execution
  - memory
  - todo
  - delegation
  - clarify

mcp_servers:
  janissary_security:
    url: "http://<janissary-host>:8081/mcp"
    transport: "http"
    description: "Sultanate security tools: appeal blocked requests, request new access"
```

Sultan can override model and tool list per province at creation time.

## Security Policy

Vizier writes these defaults to Divan when creating the province. Sentinel
reads them and provisions grants accordingly.

### Default Whitelist

Domains the province can access without restriction:

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

Sultan can expand or restrict per province.

### Default Grants

| Grant | Domain | Injection | Source |
|-------|--------|-----------|--------|
| GitHub read/write | `api.github.com`, `github.com` | `Authorization: Bearer <token>` | Sentinel provisions scoped token |

Additional grants require Sultan approval via Sentinel.

### Non-HTTP Port Declarations

| Host | Port | Protocol | Reason |
|------|------|----------|--------|
| `github.com` | 22 | TCP | Git SSH (optional, HTTPS default) |

All port openings require Sultan approval. Vizier writes declarations to
Divan, Sentinel asks Sultan, then opens on approval.

## Province Parameters (Berat-level)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `pasha_name` | no | `Pasha` | Agent's name in SOUL.md |
| `extra_instructions` | no | empty | Additional instructions appended to AGENTS.md |
| `model` | no | berat default | LLM model override |
| `extra_whitelist` | no | `[]` | Additional whitelisted domains |
| `extra_tools` | no | `[]` | Additional Hermes tools to enable |

## Phase 1 Scope

**In scope:**
- SOUL.md template for coding Pasha
- AGENTS.md template with variable substitution
- Curated tool selection (9 of 40+ Hermes tools)
- Janissary security HTTP API configuration (MCP transport wrapper)
- Default whitelist (GitHub, PyPI, npm, docs)
- Default GitHub grant (read/write, scoped token)
- Non-HTTP port declarations (Sultan approval required)
- Berat-level province parameters

**Deferred:**
- Additional berats (research-berat, assistant-berat)
- Berat inheritance (base berat + overlays)
- Berat versioning and migration
