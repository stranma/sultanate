# Technical Spec: hermes-coding-berat -- Coding Agent Profile

> Implements [HERMES_CODING_BERAT_MVP_PRD.md](HERMES_CODING_BERAT_MVP_PRD.md).
> For architecture context see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For the container template this berat runs in see
> [HERMES_FIRMAN_SPEC.md](HERMES_FIRMAN_SPEC.md).

## 1. Artifact Structure

A berat is a directory on the host at a convention path. Vizier resolves
`--berat hermes-coding-berat` to this path.

```
/opt/sultanate/berats/hermes-coding-berat/
├── berat.yaml
└── templates/
    ├── SOUL.md
    ├── AGENTS.md
    └── config.yaml
```

| File | Purpose |
|------|---------|
| `berat.yaml` | Manifest: metadata, defaults, security policy, template paths. |
| `templates/SOUL.md` | Pasha personality. Rendered to `/opt/data/SOUL.md` inside the container. |
| `templates/AGENTS.md` | Working rules. Rendered to `/opt/data/workspace/AGENTS.md` inside the container. |
| `templates/config.yaml` | Hermes runtime config. Rendered to `/opt/data/config.yaml` inside the container. |

## 2. berat.yaml Schema

```yaml
name: hermes-coding-berat
version: "0.1.0"
description: "Coding agent profile"
defaults:
  pasha_name: "Pasha"
  extra_instructions: ""
  model: "anthropic/claude-sonnet-4-20250514"
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
    - name: "github_rw"
      domain: "api.github.com"
      header: "Authorization"
      description: "GitHub API read/write"
    - name: "github_rw_web"
      domain: "github.com"
      header: "Authorization"
      description: "GitHub web read/write"
  port_requests:
    - host: "github.com"
      port: 22
      protocol: "tcp"
      reason: "Git SSH (optional)"
templates:
  soul: "templates/SOUL.md"
  instructions: "templates/AGENTS.md"
  config: "templates/config.yaml"
```

### Field Reference

#### Top-Level

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Unique berat identifier. Recorded in Divan province records (`berat` field). Sentinel reads this to look up grant templates. |
| `version` | string | yes | Semver. Vizier logs the version at province creation. |
| `description` | string | yes | Human-readable purpose. Not used at runtime. |
| `defaults` | object | yes | Default values for template variables. Sultan can override at province creation. |
| `security` | object | yes | Security policy declarations. Vizier and Sentinel process these at province creation (see §5). |
| `templates` | object | yes | Paths to template files, relative to the berat directory. |

#### defaults

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `pasha_name` | string | `"Pasha"` | Agent display name. Used in SOUL.md as `{{pasha_name}}`. Sultan can override (e.g., `"Kemal"`). |
| `extra_instructions` | string | `""` | Appended to AGENTS.md via `{{extra_instructions}}`. Sultan can pass task-specific rules. |
| `model` | string | `"anthropic/claude-sonnet-4-20250514"` | LLM model for the agent. Used in config.yaml as `{{model}}`. Sultan can override per province. |

#### security.whitelist

Array of domain strings. These are the domains the province can access without
restriction (all HTTP methods). Vizier writes these to Divan at province
creation.

#### security.grants

Array of grant **templates**. These declare _what kind_ of credential the
province needs, not the actual secret values.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique grant template name within this berat. Used as a reference key. |
| `domain` | string | Target domain for header injection. Maps to Divan grant `match.domain`. |
| `header` | string | HTTP header name to inject. Maps to Divan grant `inject.header`. |
| `description` | string | Human-readable purpose. Displayed to Sultan when Sentinel requests token provisioning. |

**Grants are templates, not secrets.** The berat declares "this province
needs an `Authorization` header for `api.github.com`." It does NOT contain
the token value. The provisioning flow:

1. Vizier writes the province to Divan (includes `berat: "hermes-coding-berat"`)
2. Sentinel polls Divan, sees the new province
3. Sentinel loads the berat's grant templates from `/opt/sultanate/berats/hermes-coding-berat/berat.yaml`
4. Sentinel asks Sultan for actual token values (or retrieves from Infisical if pre-stored)
5. Sentinel writes complete grants to Divan with real secret values:
   ```json
   {
     "province_id": "prov-a1b2c3",
     "source_ip": "172.18.0.5",
     "match": { "domain": "api.github.com" },
     "inject": { "header": "Authorization", "value": "Bearer ghp_xxxx" }
   }
   ```
6. Janissary reads the grant from Divan and injects the header at proxy time

Vizier never sees or handles secret values. The berat never contains them.

#### security.port_requests

Array of non-HTTP port declarations. These require Sultan approval before
Sentinel opens them.

| Field | Type | Description |
|-------|------|-------------|
| `host` | string | Target hostname. |
| `port` | integer | Target port number. |
| `protocol` | string | Transport protocol (`tcp` or `udp`). |
| `reason` | string | Human-readable justification. Displayed to Sultan for approval. |

Vizier writes these to Divan as pending port requests at province creation.
Sentinel reads them, relays to Sultan for approval. On approval, Sentinel
opens the specific host:port pair via iptables/Docker network rules.

#### templates

| Field | Type | Description |
|-------|------|-------------|
| `soul` | string | Path to SOUL.md template, relative to berat directory. |
| `instructions` | string | Path to AGENTS.md template, relative to berat directory. |
| `config` | string | Path to config.yaml template, relative to berat directory. |

## 3. Template Files

### templates/SOUL.md

Rendered destination: `/opt/data/SOUL.md` (inside the container).

```markdown
You are {{pasha_name}}, a Pasha in the Sultanate system. You work in
province {{province_name}}.

You execute tasks assigned by Sultan with precision and transparency.
You report progress honestly. If you're stuck, you say so. If you need
access to something, you request it through the security tool. You never
try to work around security restrictions.
```

Variables used: `{{pasha_name}}`, `{{province_name}}`.

### templates/AGENTS.md

Rendered destination: `/opt/data/workspace/AGENTS.md` (inside the container).
Hermes auto-loads this file from the workspace root as project-specific
context.

```markdown
# Working Rules

- Work only within /opt/data/workspace
- Use the security MCP tool to appeal blocked requests or request new access
- Commit to a feature branch, not main
- Create a PR when your task is complete

{{extra_instructions}}
```

Variables used: `{{extra_instructions}}`.

### templates/config.yaml

Rendered destination: `/opt/data/config.yaml` (inside the container).
Hermes reads this as its runtime configuration.

```yaml
model:
  default: "{{model}}"
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
    url: "{{janissary_url}}/mcp"
    transport: "http"
    description: "Sultanate security tools: appeal blocked requests, request new access"
```

Variables used: `{{model}}`, `{{janissary_url}}`.

## 4. Template Rendering

### Engine

Simple `str.replace()` on `{{variable}}` patterns. No logic, no loops, no
conditionals. Each `{{variable}}` in a template file is replaced with its
resolved value.

### Variable Resolution Order

For each variable, Vizier resolves the value in this order:

1. **Sultan override** -- value passed at province creation (e.g., `--pasha-name Kemal`)
2. **berat.yaml defaults** -- value from the `defaults` section
3. **System-generated** -- values Vizier generates (province_id, province_name, janissary_url)
4. **Empty string** -- if none of the above, replace with `""`

### Variable Catalog

| Variable | Required | Source | Default |
|----------|----------|--------|---------|
| `{{repo_name}}` | yes | Sultan's create command | -- (creation fails if missing) |
| `{{branch}}` | no | Sultan's create command | `main` |
| `{{province_id}}` | -- | Vizier auto-generates | `prov-<random>` |
| `{{province_name}}` | no | Sultan's create command or auto | repo name |
| `{{pasha_name}}` | no | Sultan override or berat default | `Pasha` |
| `{{extra_instructions}}` | no | Sultan override or berat default | `""` |
| `{{model}}` | no | Sultan override or berat default | `anthropic/claude-sonnet-4-20250514` |
| `{{janissary_url}}` | -- | Vizier config (Janissary's appeal API IP/port) | `http://<janissary-ip>:8081` |
| `{{workspace_dir}}` | -- | firman.yaml `workspace_dir` | `/opt/data/workspace` |

### Validation

After rendering `config.yaml`, Vizier validates that the output parses as
valid YAML. If parsing fails, province creation aborts with an error
identifying the template and the variables used. SOUL.md and AGENTS.md are
Markdown and require no parse validation.

### Example Rendering

Given Sultan's command:
```
vizier create hermes-firman --berat hermes-coding-berat --repo stranma/EFM --pasha-name Kemal
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
| `{{model}}` | `anthropic/claude-sonnet-4-20250514` |
| `{{janissary_url}}` | `http://172.18.0.2:8081` |

Rendered `/opt/data/SOUL.md`:
```markdown
You are Kemal, a Pasha in the Sultanate system. You work in
province stranma/EFM.
...
```

Rendered `/opt/data/config.yaml`:
```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"
  provider: openrouter
...
mcp_servers:
  janissary_security:
    url: "http://172.18.0.2:8081/mcp"
    transport: "http"
    ...
```

## 5. Berat Application by Vizier

After the firman's bootstrap commands complete and the entrypoint has run,
Vizier applies the berat. This is step 6 in the firman execution sequence
(see [HERMES_FIRMAN_SPEC.md §3](HERMES_FIRMAN_SPEC.md#3-how-vizier-uses-hermes-firman)).

### File Write Sequence

```
1. Render templates/SOUL.md -> write to /opt/data/SOUL.md
   (overwrites the entrypoint's default SOUL.md)

2. Render templates/AGENTS.md -> write to /opt/data/workspace/AGENTS.md
   (workspace exists from firman bootstrap repo clone)

3. Render templates/config.yaml -> write to /opt/data/config.yaml
   (overwrites the entrypoint's default config.yaml)
   -> validate YAML parse
```

Vizier writes files via `docker cp` into the container's `/opt/data` volume.

### Timing Constraint

Berat files MUST be written after:
- `docker start` (entrypoint creates default dirs and files)
- Bootstrap commands (repo clone creates `/opt/data/workspace`)

Berat files MUST be written before:
- Startup command (`hermes gateway` reads config.yaml, SOUL.md, AGENTS.md)

The firman's execution sequence (HERMES_FIRMAN_SPEC.md §3) guarantees this
ordering.

## 6. Security Policy Processing

When Vizier creates a province, it reads `berat.yaml` security section and
writes to Divan. Sentinel independently processes grants.

### Whitelist → Divan

Vizier writes the berat's default whitelist to Divan at province creation:

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

Janissary reads this per-source whitelist and allows all HTTP methods to
these domains. Sultan can later expand or restrict via Sentinel.

### Grants → Sentinel → Divan

**Vizier does NOT write grants to Divan.** The flow:

1. Vizier writes the province record to Divan:
   ```
   POST /provinces
   { "id": "prov-a1b2c3", ..., "berat": "hermes-coding-berat" }
   ```

2. Sentinel polls Divan, sees new province with `berat: "hermes-coding-berat"`

3. Sentinel reads `/opt/sultanate/berats/hermes-coding-berat/berat.yaml`,
   extracts `security.grants`:
   ```yaml
   grants:
     - name: "github_rw"
       domain: "api.github.com"
       header: "Authorization"
       description: "GitHub API read/write"
     - name: "github_rw_web"
       domain: "github.com"
       header: "Authorization"
       description: "GitHub web read/write"
   ```

4. Sentinel retrieves actual token values from Infisical (pre-stored by
   Sultan) or asks Sultan via Telegram for the token

5. Sentinel writes complete grants to Divan:
   ```
   POST /grants
   {
     "province_id": "prov-a1b2c3",
     "source_ip": "172.18.0.5",
     "match": { "domain": "api.github.com" },
     "inject": { "header": "Authorization", "value": "Bearer ghp_xxxx" }
   }
   ```
   (One POST per grant template, with the real secret value.)

6. Janissary reads grants from Divan and injects headers at proxy time

This separation ensures Vizier never handles dangerous secrets. Only Sentinel
(trusted, root) and Janissary (reads from Divan) see token values.

### Port Requests → Divan

Vizier writes each berat port declaration to Divan as a pending request:

```
POST /port_requests
{
  "province_id": "prov-a1b2c3",
  "host": "github.com",
  "port": 22,
  "protocol": "tcp",
  "reason": "Git SSH (optional)"
}
```

Sentinel reads pending port requests, relays to Sultan for approval. On
approval, Sentinel opens the host:port pair via iptables/Docker network
rules. No auto-approve.

## 7. Janissary MCP Server Configuration

The Janissary MCP server is an HTTP API served by Janissary, not an npm
package. Provinces connect to it over the internal Docker network.

### config.yaml mcp_servers Section

```yaml
mcp_servers:
  janissary_security:
    url: "{{janissary_url}}/mcp"
    transport: "http"
    description: "Sultanate security tools: appeal blocked requests, request new access"
```

| Field | Value | Description |
|-------|-------|-------------|
| `url` | `{{janissary_url}}/mcp` | Janissary's MCP endpoint on the internal network. `{{janissary_url}}` is injected by Vizier (e.g., `http://172.18.0.2:8081`). |
| `transport` | `http` | HTTP-based MCP transport. No stdio, no npx. |
| `description` | (string) | Shown to the agent as tool documentation. |

### Why HTTP, Not npx

The PRD shows `command: "npx"` with `@sultanate/janissary-mcp`. The actual
implementation uses HTTP transport because:

1. Janissary is already an HTTP server on the internal network
2. No npm package install required inside the container
3. No Node.js process spawned per-tool-call
4. Janissary can authenticate the MCP caller by source IP (same as proxy traffic)

### MCP Tools Provided

Two tools are available via the MCP server (see
[JANISSARY_MVP_PRD.md](JANISSARY_MVP_PRD.md)):

**`appeal_request`** -- appeal a blocked outbound request:
```
appeal_request(url: string, method: string, justification: string)
-> { status: "pending" }
```
Janissary writes the appeal to Divan. Vizier polls and relays to Sultan.

**`request_access`** -- request new credentials or port access:
```
request_access(service: string, scope: string, justification: string)
-> { status: "pending" }
```
Routed to Sultan via Vizier. Sultan tells Sentinel to provision.

### Janissary URL Injection

Vizier sets `{{janissary_url}}` during template rendering. The value is
Janissary's address on the internal Docker network (e.g.,
`http://172.18.0.2:8081`). This is NOT the proxy port (8080) -- it's the
`HTTPS_PROXY` env vars, but the MCP endpoint is a distinct path (`/mcp`).

## 8. Cross-Reference: Firman ↔ Berat Boundary

| Concern | Firman (hermes-firman) | Berat (hermes-coding-berat) |
|---------|----------------------|----------------------------|
| Docker image | Declares it (`image` field) | Does not reference it |
| Repo clone | Bootstrap commands | Does not reference it |
| Workspace path | Declares `workspace_dir` | Uses it in templates (`/opt/data/workspace`) |
| SOUL.md | Does not provide it | Provides template |
| AGENTS.md | Does not provide it | Provides template |
| config.yaml | Does not provide it | Provides template |
| Proxy env vars | Not declared (Vizier sets) | Not declared (Vizier sets) |
| CA cert | Not declared (Vizier handles) | Not declared (Vizier handles) |
| Whitelist | Not declared | Declares defaults |
| Grants | Not declared | Declares templates (no secrets) |
| Port requests | Not declared | Declares defaults |
| Startup command | Declares it (`startup` field) | Does not reference it |
| Template variables | Uses `{{repo_name}}`, `{{branch}}`, `{{workspace_dir}}` | Uses all variables (§4) |
