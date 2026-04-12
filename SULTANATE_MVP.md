# Sultanate MVP -- Secure Agent Deployment Platform

## What This Is

Run AI agents in isolated containers with controlled internet access and
conversational management. Sultan (you) talks to agents via Telegram, agents
work in sandboxed Docker containers, and no dangerous secret ever enters a
container.

## MVP Scope

Single operator, single host, Hermes-native. One firman (hermes-firman), one
berat (hermes-coding-berat). Conversational management via Telegram + CLI.

## Architecture

```
Sultan (Telegram)
  |
  +-- Vizier (Hermes agent, vizier user, Docker group)
  |     Job: province lifecycle, Docker management, appeal relay
  |     Egress: through Janissary
  |     Tools: vizier CLI via terminal
  |
  +-- Sentinel (Hermes agent, root)
  |     Job: secret management, grant provisioning
  |     Egress: Telegram API + Infisical (local) only
  |     Tools: infisical CLI, Divan API
  |
  +-- Divan (shared state, SQLite + HTTP API)
  |     Province registry, grants, whitelists, blacklist, appeals
  |     Written by Vizier and Sentinel, read by Janissary
  |
  +-- Janissary (transparent proxy via WireGuard, dumb)
  |     Reads all state from Divan
  |     Whitelist: pass all traffic
  |     Non-whitelist: read-only (GET/HEAD pass, writes blocked)
  |     Blocked + appeal: written to Divan, Vizier relays to Sultan
  |     Credential injection: reads grants from Divan
  |
  +-- Provinces (Docker containers, internal network only)
        HTTP/HTTPS: through Janissary
        Non-HTTP: blocked by default, specific host:port pairs
                  whitelisted per-province via Docker network rules
```

## Trust Model

| Trust Level | Component | Access |
|-------------|-----------|--------|
| **Trusted** | Sentinel (root) | Full host access. Manages secrets. Knows the policy: dangerous secrets go through Janissary, never into containers. |
| **Semi-trusted** | Vizier (vizier user, Docker group) | Creates/manages containers, execs into provinces. Cannot read grant files (filesystem permissions). |
| **Untrusted** | Province / Pasha | Sandboxed container. No direct internet. All HTTP through Janissary. Never holds dangerous secrets. |

Sentinel is trusted and instructed, not constrained. It has root but follows
policy -- same as a sysadmin.

## Credential Model

**Dangerous secrets** (GitHub tokens, API keys) -- Sentinel provisions via
Infisical, writes grants to Divan. Janissary reads them from Divan and
injects into request headers at proxy level. Containers never see these
values.

**Low-risk config** (Telegram bot tokens, public endpoints) -- Vizier writes
directly into containers. A leaked bot token lets someone chat as the agent,
not access code.

## CA Certificate Lifecycle

A Sultanate-wide CA certificate is generated once at deploy time. The CA cert
is installed in all province containers so mitmproxy can decrypt and inspect
HTTPS traffic. The CA private key is only accessible to the Janissary
container.

## Divan (Shared State)

Minimal shared state store: SQLite + lightweight HTTP API. No web dashboard,
no audit queries -- just a registry that all components read/write through
REST.

| Endpoint | Writer | Reader | Data |
|----------|--------|--------|------|
| `/provinces` | Vizier | Janissary, Sentinel | ID, IP, name, status, firman, berat |
| `/grants` | Sentinel | Janissary | Source IP + domain -> header injection |
| `/whitelists` | Vizier (berat defaults at creation), Sentinel (Sultan's changes) | Janissary | Per-source allowed domains |
| `/blacklist` | Sentinel (on Sultan's instruction) | Janissary | Global blocked domains |
| `/appeals` | Janissary | Vizier | Pending/resolved appeal records |

**Access control:** Divan authenticates callers via pre-shared API keys (one
per component, generated at deploy time). Each key maps to a role with
endpoint-level read/write permissions. Grant secret values are only returned
to the Janissary role. See DIVAN_API_SPEC.md for details.

Divan ships with Janissary in the same repo. It starts before all other
components.

## Artifact Formats

**Firman** (container template) -- a directory containing a `firman.yaml`
manifest and optional supporting files. Stored at a convention path on the
host (`/opt/sultanate/firmans/<name>/`). Vizier resolves `--firman <name>`
by looking up this path. See HERMES_FIRMAN_MVP_PRD.md.

**Berat** (agent profile) -- a directory containing a `berat.yaml` manifest
and template files (SOUL.md, AGENTS.md, config.yaml). Stored at
`/opt/sultanate/berats/<name>/`. Vizier resolves `--berat <name>` by
looking up this path. Templates use `{{variable}}` syntax with simple string
substitution. See HERMES_CODING_BERAT_MVP_PRD.md.

## GitHub Token Strategy

Sultan pre-creates GitHub PATs (fine-grained, scoped per repo). Sultan gives
them to Sentinel via Telegram: "Store this token for repo X." Sentinel stores
in Infisical and writes the grant to Divan. Sentinel does not create tokens
programmatically -- it stores and manages tokens that Sultan provides.

## Runtime

All Hermes agents (Vizier, Sentinel, province Pashas) use the upstream
`nousresearch/hermes-agent` Docker image. No custom Dockerfile. Each agent
is differentiated by its configuration (soul, tools, MCP servers, env vars)
applied at startup. See [HERMES_FIRMAN_MVP_PRD.md](HERMES_FIRMAN_MVP_PRD.md).

## Startup Order

Janissary must start before Vizier. If Janissary is down, Vizier cannot reach
Telegram -- Sultan loses the management channel. This is fail-closed by
design. Sultan has SSH to the host as fallback.

```
1. Divan (shared state)
2. Janissary (proxy, reads from Divan)
3. Sentinel (secrets, writes to Divan, direct networking)
4. Vizier (management, writes to Divan, through Janissary)
5. Provinces (on demand, through Janissary)
```

## Network Model

- Provinces on internal Docker network (`internal: true`, no external route)
- All province HTTP/HTTPS goes through Janissary (WireGuard transparent proxy)
- Non-HTTP access: blocked by default. Berat declares needed ports, Vizier
  writes them to Divan as requests, Sentinel asks Sultan for approval, then
  opens specific host:port pairs (iptables/Docker rules) and provisions
  service tokens. No auto-approve.
- Vizier's traffic goes through Janissary (same rules as provinces)
- Vizier has the Janissary security MCP tool -- can appeal blocks or ask
  Sultan to tell Sentinel to add permanent whitelist exceptions
- Sentinel has direct host networking (needs Telegram + Infisical only)

## Traffic Rules (Janissary)

1. **Blacklist** -- domain on global blacklist? Block all traffic. Overrides whitelist.
2. **Whitelist** -- domain on province's allowlist? Pass all traffic.
3. **Non-whitelist read** -- GET/HEAD to unknown domain? Pass (browsing, docs).
4. **Non-whitelist write** -- POST/PUT/PATCH/DELETE to unknown domain? Block.
5. **Appeal** -- agent appeals a block with justification. Stored by Janissary,
   relayed by Vizier to Sultan via Telegram. Sultan approves (one-time or
   whitelist) or denies.

No size gate, no Kashif, no automated triage. Sultan decides.

## Appeal Flow

1. Agent's write request to non-whitelisted domain -> blocked by Janissary
2. Agent calls `appeal_request(url, method, justification)` via MCP tool
3. Janissary writes appeal to Divan
4. Vizier polls Divan for pending appeals, relays to Sultan via Telegram
5. Sultan replies: approve / deny / whitelist
6. Vizier writes decision to Divan, Janissary picks it up

## Components

| Component | Repo | Description |
|-----------|------|-------------|
| **Vizier** | `vizier` | CLI + Hermes agent. Province lifecycle management. |
| **Janissary** | `janissary` | Whitelist proxy + credential injection. |
| **Divan** | `janissary` | SQLite + HTTP API. Shared state store. Ships with Janissary. |
| **Sentinel** | `janissary` | Secret management Hermes agent. Ships with Janissary. |
| **hermes-firman** | `hermes-firman` | Docker image + bootstrap for Hermes provinces. |
| **hermes-coding-berat** | `hermes-coding-berat` | Coding agent profile (soul, tools, whitelist, grants). |

## What's Deferred

- Kashif (LLM content inspector) -- no automated appeal triage
- Web dashboard -- CLI and Telegram only
- SentinelGate (MCP-level security) -- no tool-level RBAC
- Session tracking and behavioral analysis
- Multi-firman, multi-berat support
- Cross-province coordination
- Signed audit records
- Multi-operator support
