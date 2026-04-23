# Technical Spec: openclaw-coding-berat -- Coding Agent Profile

> Implements [OPENCLAW_CODING_BERAT_MVP_PRD.md](OPENCLAW_CODING_BERAT_MVP_PRD.md).
> For architecture context see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For the container template this berat runs in see
> [OPENCLAW_FIRMAN_SPEC.md](OPENCLAW_FIRMAN_SPEC.md).

## 1. Artifact Structure

A berat is a directory on the host at a convention path. Vizier resolves
`--berat openclaw-coding-berat` to this path.

```
/opt/sultanate/berats/openclaw-coding-berat/
├── berat.yaml
└── templates/
    ├── SOUL.md
    ├── AGENTS.md
    ├── IDENTITY.md
    └── openclaw.json
```

| File | Purpose |
|------|---------|
| `berat.yaml` | Manifest: metadata, defaults, security policy, template paths. |
| `templates/SOUL.md` | Pasha personality. Rendered to `/opt/data/workspace/SOUL.md`. |
| `templates/AGENTS.md` | Working rules. Rendered to `/opt/data/workspace/AGENTS.md`. |
| `templates/IDENTITY.md` | Agent identity / emoji. Rendered to `/opt/data/workspace/IDENTITY.md`. Optional. |
| `templates/openclaw.json` | OpenClaw runtime config. Rendered to `/opt/data/.openclaw/openclaw.json`. |

## 2. berat.yaml Schema

```yaml
name: openclaw-coding-berat
version: "0.1.0"
description: "Coding agent profile"
defaults:
  pasha_name: "Pasha"
  extra_instructions: ""
  model: "anthropic/claude-sonnet-4"
security:
  whitelist:
    - github.com
    - api.github.com
    - pypi.org
    - files.pythonhosted.org
    - registry.npmjs.org
    - cdn.jsdelivr.net
    - docs.python.org
    - stackoverflow.com
  grants:
    - name: "github_app"
      service: "github"
      kind: "dynamic"
      domains:
        - "api.github.com"
        - "github.com"
      header: "Authorization"
      description: "GitHub App installation token (minted by Aga, 1-hour TTL, auto-renewed)"
  port_requests:
    - host: "github.com"
      port: 22
      protocol: "tcp"
      reason: "Git SSH (optional)"
templates:
  soul:     "templates/SOUL.md"
  agents:   "templates/AGENTS.md"
  identity: "templates/IDENTITY.md"
  config:   "templates/openclaw.json"
```

### Field Reference

#### Top-Level

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Unique berat identifier. Recorded in Divan province records (`berat` field). Aga reads this to look up grant templates. |
| `version` | string | yes | Semver. Vizier logs the version at province creation. |
| `description` | string | yes | Human-readable purpose. Not used at runtime. |
| `defaults` | object | yes | Default values for template variables. Sultan can override at province creation. |
| `security` | object | yes | Security policy declarations. Vizier and Aga process these at province creation (see §5). |
| `templates` | object | yes | Paths to template files, relative to the berat directory. |

#### defaults

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `pasha_name` | string | `"Pasha"` | Agent display name. Used in SOUL.md as `{{pasha_name}}`. Sultan can override (e.g., `"Kemal"`). |
| `extra_instructions` | string | `""` | Appended to AGENTS.md via `{{extra_instructions}}`. Sultan can pass task-specific rules. |
| `model` | string | `"anthropic/claude-sonnet-4"` | LLM model for the agent. Used in `openclaw.json` as `{{model}}`. Sultan can override per province. Format: `"provider/model-id"`. |

#### security.whitelist

Array of domain strings. These are the domains the province can access
without restriction (all HTTP methods). Vizier writes these to Divan at
province creation.

#### security.grants

Array of grant **templates**. These declare _what kind_ of credential
the province needs, not the actual secret values.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique grant template name within this berat. Used as a reference key. |
| `service` | string | Logical service name (e.g. `github`). Aga uses this to dispatch to the right mint path. |
| `kind` | string | `dynamic` (Aga mints via OpenBao dynamic engine -- for `github`, GitHub App installation tokens) or `kv` (Sultan-pasted, Aga stores in OpenBao KV with no lease). |
| `domains` | list[string] | Target domains for header injection. One Divan grant record is written per domain, all sharing the same token value and lease metadata. |
| `header` | string | HTTP header name to inject. Maps to Divan grant `inject.header`. |
| `description` | string | Human-readable purpose. Displayed to Sultan in Aga's alerts. |

**Grants are templates, not secrets.** The berat declares "this province
needs an `Authorization` header for `api.github.com`, kind=dynamic,
service=github." It does NOT contain the token value. The provisioning
flow:

1. Vizier writes the province to Divan (includes
   `berat: "openclaw-coding-berat"`)
2. Aga polls Divan, sees the new province.
3. Aga loads the berat's grant templates from
   `/opt/sultanate/berats/openclaw-coding-berat/berat.yaml`.
4. For `kind: "dynamic"` + `service: "github"`: Aga reads the GitHub
   App private key from OpenBao KV, mints an installation token
   scoped to the province's repo, receives `{token, expires_at}`
   (1-hour TTL).
5. Aga writes one grant record to Divan per domain in the `domains`
   list, all referencing the same token + lease:
   ```json
   {
     "province_id":      "prov-a1b2c3",
     "source_ip":        "10.13.13.5",
     "match":  { "domain": "api.github.com" },
     "inject": { "header": "Authorization",
                 "value":  "Bearer <token>" },
     "openbao_lease_id": "github-app:prov-a1b2c3",
     "lease_expires_at": "<token-expiry>"
   }
   ```
6. Aga's renewal loop refreshes these grants before expiry while the
   province is `running`.
7. Janissary reads grants from Divan and injects headers at proxy
   time, checking `lease_expires_at` before each injection.

Vizier never sees or handles secret values. The berat never contains
them.

#### security.port_requests

Array of non-HTTP port declarations. These require Sultan approval
before Aga opens them.

| Field | Type | Description |
|-------|------|-------------|
| `host` | string | Target hostname. |
| `port` | integer | Target port number. |
| `protocol` | string | Transport protocol (`tcp` or `udp`). |
| `reason` | string | Human-readable justification. Displayed to Sultan for approval. |

Vizier writes these to Divan as pending port requests at province
creation. Aga reads them, relays to Sultan for approval. On approval,
Aga opens the specific host:port pair via iptables. No auto-approve.

#### templates

| Field | Type | Description |
|-------|------|-------------|
| `soul` | string | Path to SOUL.md template, relative to berat directory. |
| `agents` | string | Path to AGENTS.md template, relative to berat directory. |
| `identity` | string | Path to IDENTITY.md template (optional). |
| `config` | string | Path to `openclaw.json` template, relative to berat directory. |

## 3. Template Files

### templates/SOUL.md

Rendered destination: `/opt/data/workspace/SOUL.md` (inside the
container). OpenClaw auto-loads this at first session turn.

```markdown
You are {{pasha_name}}, a Pasha in the Sultanate system. You work in
province {{province_name}} on repository {{repo_name}}.

You execute tasks assigned by Sultan with precision and transparency.
You report progress honestly. If you're stuck, you say so. If you need
access to something outside your whitelist, you request it through the
janissary_security MCP tool -- never by trying to work around the
proxy. If a request is blocked and the block seems wrong, appeal it
with a clear justification. Be direct. Be honest.
```

Variables used: `{{pasha_name}}`, `{{province_name}}`, `{{repo_name}}`.

### templates/AGENTS.md

Rendered destination: `/opt/data/workspace/AGENTS.md` (inside the
container). OpenClaw auto-loads this as project-specific context.

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

Variables used: `{{extra_instructions}}`.

### templates/IDENTITY.md

Rendered destination: `/opt/data/workspace/IDENTITY.md`. Optional.

```markdown
{{pasha_name}} 👷
```

Variables used: `{{pasha_name}}`.

### templates/openclaw.json

Rendered destination: `/opt/data/.openclaw/openclaw.json` (inside the
container). OpenClaw reads this as its runtime configuration.

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

Variables used: `{{model}}`, `{{pasha_telegram_bot_token}}`,
`{{sultan_telegram_user_id}}`, `{{janissary_api}}`.

## 4. Template Rendering

### Engine

Simple `str.replace()` on `{{variable}}` patterns. No logic, no loops,
no conditionals. Each `{{variable}}` in a template file is replaced
with its resolved value.

### Variable Resolution Order

For each variable, Vizier resolves the value in this order:

1. **Sultan override** -- value passed at province creation (e.g.,
   `--pasha-name Kemal`)
2. **berat.yaml defaults** -- value from the `defaults` section
3. **System-generated** -- values Vizier generates (province_id,
   province_name, pasha_telegram_bot_token, janissary_api)
4. **Deploy-time environment** -- values from the deploy-time
   environment (sultan_telegram_user_id)
5. **Empty string** -- if none of the above, replace with `""`

### Variable Catalog

| Variable | Required | Source | Default |
|----------|----------|--------|---------|
| `{{repo_name}}` | yes | Sultan's create command | -- (creation fails if missing) |
| `{{branch}}` | no | Sultan's create command | `main` |
| `{{province_id}}` | -- | Vizier auto-generates | `prov-<random>` |
| `{{province_name}}` | no | Sultan's create command or auto | repo name |
| `{{pasha_name}}` | no | Sultan override or berat default | `Pasha` |
| `{{extra_instructions}}` | no | Sultan override or berat default | `""` |
| `{{model}}` | no | Sultan override or berat default | `anthropic/claude-sonnet-4` |
| `{{pasha_telegram_bot_token}}` | -- | Vizier (bot pool) | -- |
| `{{sultan_telegram_user_id}}` | -- | Deploy env (`SULTAN_TELEGRAM_USER_ID`) | -- |
| `{{janissary_api}}` | -- | Vizier config (Janissary's appeal API URL) | `http://10.13.13.1:8081` |
| `{{workspace_dir}}` | -- | firman.yaml `workspace_dir` | `/opt/data/workspace` |

### Validation

After rendering `openclaw.json`, Vizier validates that the output
parses as valid JSON. If parsing fails, province creation aborts with
an error identifying the template and the variables used. SOUL.md,
AGENTS.md, and IDENTITY.md are Markdown and require no parse validation.

### Example Rendering

Given Sultan's command:

```
vizier-cli create openclaw-firman --berat openclaw-coding-berat \
  --repo stranma/EFM --pasha-name Kemal
```

Variables resolve to:

| Variable | Value |
|----------|-------|
| `{{repo_name}}` | `stranma/EFM` |
| `{{branch}}` | `main` |
| `{{province_id}}` | `prov-a1b2c3` |
| `{{province_name}}` | `stranma/EFM` |
| `{{pasha_name}}` | `Kemal` |
| `{{extra_instructions}}` | `""` |
| `{{model}}` | `anthropic/claude-sonnet-4` |
| `{{pasha_telegram_bot_token}}` | `123456:ABC-def...` |
| `{{sultan_telegram_user_id}}` | `123456789` |
| `{{janissary_api}}` | `http://10.13.13.1:8081` |

Rendered `/opt/data/workspace/SOUL.md`:

```markdown
You are Kemal, a Pasha in the Sultanate system. You work in
province stranma/EFM on repository stranma/EFM.
...
```

Rendered `/opt/data/.openclaw/openclaw.json`:

```json
{
  "agent": {
    "model":     "anthropic/claude-sonnet-4",
    "workspace": "/opt/data/workspace"
  },
  ...
  "mcp_servers": {
    "janissary_security": {
      "transport": "http",
      "url":       "http://10.13.13.1:8081/mcp"
    }
  }
}
```

## 5. Berat Application by Vizier

After the firman's bootstrap commands complete (including repo clone)
and the entrypoint has run, Vizier applies the berat. This is step 7
in the firman execution sequence (see
[OPENCLAW_FIRMAN_SPEC.md §3](OPENCLAW_FIRMAN_SPEC.md#3-how-vizier-uses-openclaw-firman)).

### File Write Sequence

```
0. Ensure target directories exist:
   docker exec <container> mkdir -p /opt/data/.openclaw /opt/data/workspace

1. Render templates/SOUL.md -> write to /opt/data/workspace/SOUL.md
2. Render templates/AGENTS.md -> write to /opt/data/workspace/AGENTS.md
3. (if present) Render templates/IDENTITY.md -> write to
   /opt/data/workspace/IDENTITY.md
4. Render templates/openclaw.json -> write to
   /opt/data/.openclaw/openclaw.json
   -> validate JSON parse
5. chown -R <openclaw-user>:<openclaw-user> /opt/data/workspace
   /opt/data/.openclaw
   (so the non-root runtime user can read the files)
```

Vizier writes files via `docker exec -i ... tee`.

### Timing Constraint

Berat files MUST be written after:

- `docker start` (entrypoint creates default directories and
  permissions)
- Bootstrap commands (repo clone populates `/opt/data/workspace`)

Berat files MUST be written before:

- Startup command (`openclaw gateway` reads openclaw.json and the
  workspace-root auto-loaded files)

The firman's execution sequence (OPENCLAW_FIRMAN_SPEC.md §3) guarantees
this ordering.

## 6. Security Policy Processing

When Vizier creates a province, it reads `berat.yaml` security section
and writes to Divan. Aga independently processes grants.

### Whitelist → Divan

Vizier writes the berat's default whitelist to Divan at province
creation:

```
PUT /whitelists/{province_id}
{
  "domains": [
    "github.com",
    "api.github.com",
    "pypi.org",
    "files.pythonhosted.org",
    "registry.npmjs.org",
    "cdn.jsdelivr.net",
    "docs.python.org",
    "stackoverflow.com"
  ]
}
```

Janissary reads this per-source whitelist and allows all HTTP methods
to these domains. Sultan can later expand or restrict via Aga.

### Grants → Aga → Divan

**Vizier does NOT write grants to Divan.** The flow:

1. Vizier writes the province record to Divan:

   ```
   POST /provinces
   { "id": "prov-a1b2c3", ..., "berat": "openclaw-coding-berat",
     "repo": "stranma/EFM" }
   ```

2. Aga polls Divan, sees new province with
   `berat: "openclaw-coding-berat"`.

3. Aga reads
   `/opt/sultanate/berats/openclaw-coding-berat/berat.yaml`, extracts
   `security.grants`:

   ```yaml
   grants:
     - name:    "github_app"
       service: "github"
       kind:    "dynamic"
       domains: [ "api.github.com", "github.com" ]
       header:  "Authorization"
       description: "GitHub App installation token ..."
   ```

4. For `kind: "dynamic"` + `service: "github"`: Aga reads the GitHub
   App private key from OpenBao KV, mints an installation token
   scoped to `repo` (the province's repo), receives
   `{token, expires_at}`.

5. Aga writes one Divan grant per domain, all sharing the same
   token + lease metadata:

   ```
   POST /grants
   {
     "province_id": "prov-a1b2c3",
     "source_ip":   "10.13.13.5",
     "match":       { "domain": "api.github.com" },
     "inject":      { "header": "Authorization",
                      "value":  "Bearer <token>" },
     "openbao_lease_id": "github-app:prov-a1b2c3",
     "lease_expires_at": "<token-expiry from GitHub>"
   }
   ```

6. Aga's renewal loop refreshes these grants before expiry while the
   province is `running` (see `AGA_SPEC.md` §4).

7. Janissary reads grants from Divan and injects headers at proxy
   time, checking `lease_expires_at` before each injection.

This separation ensures Vizier never handles dangerous secrets. Only
Aga (trusted, root, sole OpenBao client) and Janissary (reads from
Divan grants) see token values.

### Port Requests → Divan

Vizier writes each berat port declaration to Divan as a pending
request:

```
POST /port_requests
{
  "province_id": "prov-a1b2c3",
  "host":        "github.com",
  "port":        22,
  "protocol":    "tcp",
  "reason":      "Git SSH (optional)"
}
```

Aga reads pending port requests, relays to Sultan for approval. On
approval, Aga opens the host:port pair via iptables. No auto-approve.

## 7. Janissary HTTP API Configuration

Janissary exposes an HTTP API for appeals and access requests.
Provinces connect to it over the WireGuard tunnel. OpenClaw's MCP
servers feature registers this HTTP API as a tool provider.

### openclaw.json mcp_servers Section

```json
"mcp_servers": {
  "janissary_security": {
    "transport": "http",
    "url":       "{{janissary_api}}/mcp"
  }
}
```

| Field | Value | Description |
|-------|-------|-------------|
| `transport` | `http` | HTTP-based MCP transport. No stdio, no npx. |
| `url` | `{{janissary_api}}/mcp` | Janissary's MCP endpoint. `{{janissary_api}}` is injected by Vizier (e.g., `http://10.13.13.1:8081`). |

### Why HTTP, Not stdio

OpenClaw supports both stdio MCP (subprocess + stdin/stdout) and HTTP
MCP. We use HTTP because:

1. Janissary is already an HTTP server on the internal network.
2. No subprocess spawn per tool call.
3. Janissary authenticates the MCP caller by source IP (same as proxy
   traffic).

### MCP Tools Provided

Two tools are available via the HTTP API (see
[JANISSARY_MVP_PRD.md](JANISSARY_MVP_PRD.md) and
[JANISSARY_SPEC.md](JANISSARY_SPEC.md)):

**`appeal_request`** -- appeal a blocked outbound request:

```
appeal_request(url: string, method: string,
               payload: string, justification: string)
-> { status: "pending", appeal_id: "appeal-..." }
```

Janissary writes the appeal to Divan and forwards payload + justification
to Kashif for triage. Verdict is eventually written to Divan by Kashif
(auto-allow/auto-block) or by Sultan via Vizier (escalate).

**`request_access`** -- request new credentials or port access:

```
request_access(service: string, scope: string, justification: string)
-> { status: "pending" }
```

Text is Kashif-screened before reaching Aga's LLM context. Routed to
Sultan via Vizier. Sultan tells Aga to provision.

### Janissary API Injection

Vizier sets `{{janissary_api}}` during template rendering. The value is
Janissary's address on the WireGuard tunnel
(`http://10.13.13.1:8081`). The `/mcp` path on this port serves the
MCP tools.

## 8. Cross-Reference: Firman ↔ Berat Boundary

| Concern | Firman (openclaw-firman) | Berat (openclaw-coding-berat) |
|---------|-------------------------|------------------------------|
| Docker image | Declares it (`image` field) | Does not reference it |
| Repo clone | Bootstrap commands | Does not reference it |
| Workspace path | Declares `workspace_dir` | Uses it in templates (`/opt/data/workspace`) |
| OPENCLAW_HOME | Declares `openclaw_home` | Does not reference it (implicit via openclaw.json location) |
| SOUL.md | Does not provide it | Provides template |
| AGENTS.md | Does not provide it | Provides template |
| IDENTITY.md | Does not provide it | Provides template (optional) |
| openclaw.json | Does not provide it | Provides template |
| WireGuard routing | Not declared (transparent) | Not declared (transparent) |
| CA cert | Not declared (Vizier handles) | Not declared (Vizier handles) |
| Whitelist | Not declared | Declares defaults |
| Grants | Not declared | Declares templates (no secrets); `kind: dynamic` vs `kv` determines Aga's mint path |
| Port requests | Not declared | Declares defaults |
| Startup command | Declares it (`startup` field: `openclaw gateway --port 18789`) | Does not reference it |
| Template variables | Uses `{{repo_name}}`, `{{branch}}`, `{{workspace_dir}}` | Uses all variables (§4) |
| Sandbox mode | Not declared | Sets `sandbox.mode: "off"` in openclaw.json (outer container is the boundary) |
