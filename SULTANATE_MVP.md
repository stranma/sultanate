# Sultanate MVP -- Secure Agent Deployment Platform

## What This Is

Run AI agents in isolated containers with controlled internet access and
conversational management. Sultan (you) talks to agents via Telegram, agents
work in sandboxed Docker containers, and no dangerous secret ever enters a
container.

## MVP Scope

Single operator, single host, OpenClaw-native. One firman
(`openclaw-firman`), one berat (`openclaw-coding-berat`). Conversational
management via Telegram + CLI.

**Target host:** Hetzner AX41-NVMe (AMD Ryzen 5 3600, 6c/12t, 64 GB DDR4,
2x512 GB NVMe, no GPU). Kashif models run on CPU; Aga and Pashas call
Anthropic/OpenRouter via Janissary for LLM inference.

**Core RAM budget:** ~6-10 GB across OpenBao + Divan + Janissary + Kashif
+ Aga + Vizier + Docker overhead. Leaves ~50 GB for provinces (5-10
concurrent coding agents comfortable).

## Threat Model

**In scope:** a hostile or compromised Pasha (agent inside a province). We
defend hard here: network-level isolation, deterministic proxy rules,
content screening, credential injection outside the container.

**Out of scope (MVP):** compromised Aga, compromised Janissary, compromised
Kashif, compromised host, compromised OpenBao. These are all trusted-core
and assumed uncompromised. Hardening against trusted-core compromise is a
Phase 2 concern.

## Architecture

```
Sultan (Telegram)
  |
  +-- Vizier (OpenClaw agent, vizier user, Docker group)
  |     Job: province lifecycle, Docker management, appeal relay
  |     Egress: through Janissary
  |     Tools: vizier CLI via bash
  |
  +-- Aga (OpenClaw agent, root)
  |     Job: secret management, grant provisioning, alert
  |          contextualization, access-request review
  |     Egress: Telegram API + OpenBao (local 127.0.0.1) only
  |     Tools: OpenBao client, Divan API, iptables
  |     Ingress: all Pasha-originated content and fetched web pages
  |              pre-screened by Kashif
  |
  +-- Divan (shared state, SQLite + FastAPI + read-only dashboard)
  |     Province registry, grants (with OpenBao lease IDs), whitelists,
  |     blacklist, appeals, port_requests, audit log
  |     Written by Vizier and Aga, read by Janissary and dashboard
  |     Dashboard: port 8601, Jinja2 + HTMX, HTTP basic auth,
  |                bound to 127.0.0.1 (Sultan reaches via SSH tunnel)
  |
  +-- Janissary (transparent proxy via WireGuard, dumb)
  |     Reads all state from Divan
  |     Whitelist: pass all traffic
  |     Non-whitelist: read-only (GET/HEAD pass, writes blocked)
  |     Blocked + appeal: forwarded to Kashif for triage; unclear
  |                       cases escalate to Aga and Sultan
  |     Credential injection: reads grants from Divan; skips if
  |                           OpenBao lease expired (fail closed)
  |
  +-- Kashif (paranoid content inspector, local LLM on host)
  |     Job: screen appeal payloads and Aga ingress for malice /
  |          prompt injection
  |     Models (all CPU): LLM Guard regex scanners + Prompt Guard 2
  |                       22M + Llama Guard 3 1B Q4
  |     Endpoints: POST /screen/appeal, POST /screen/ingress
  |     Fail-closed: timeout or LLM down -> block + alert Sultan
  |
  +-- OpenBao (Secret Vault, single Go binary, local)
  |     Holds dangerous secrets (GitHub App tokens, DB creds,
  |     SSH CA, PKI certs). Dynamic secret engines where possible.
  |     Every credential issued with a lease (TTL-bounded).
  |     Listener bound to 127.0.0.1 only; Aga is the sole client.
  |     Manual unseal by Sultan at boot.
  |
  +-- Provinces (Docker containers, internal network only)
        HTTP/HTTPS: through Janissary
        Non-HTTP: blocked by default, specific host:port pairs
                  whitelisted per-province via Docker network rules
        Agent runtime: OpenClaw (open AI agent runtime, MIT,
                       multi-provider: Claude/OpenAI/OpenRouter/...).
                       Gateway daemon on port 18789.
```

## Trust Model

| Trust Level | Component | Access |
|-------------|-----------|--------|
| **Trusted** | Aga (root) | Full host access. Sole OpenBao client. Manages secrets. All LLM-context ingress pre-screened by Kashif. |
| **Trusted** | Janissary (root) | Network enforcement. Reads Divan, applies deterministic rules. No LLM, no outbound of its own. |
| **Trusted** | Kashif (runs alongside Janissary) | Local LLM only. Screens appeal payloads and Aga ingress. No secrets, no outbound. |
| **Trusted** | OpenBao (local, 127.0.0.1) | Holds dangerous secrets. Only Aga can authenticate. |
| **Semi-trusted** | Vizier (vizier user, Docker group) | Creates/manages containers, execs into provinces. Cannot read grant files (filesystem permissions). |
| **Untrusted** | Province / Pasha | Sandboxed container. No direct internet. All HTTP through Janissary. Never holds dangerous secrets. |

Aga is trusted and instructed, not constrained. It has root but follows
policy -- same as a sysadmin.

## Credential Model

**Dangerous secrets** (GitHub tokens, API keys) -- Aga stores in OpenBao
and writes a grant record to Divan. Janissary reads grants from Divan
and injects into request headers at the proxy level. Containers never
see these values.

Two storage modes (OpenBao engines):

- **KV mode (Phase 1 default, Sultan-pasted PATs):** Aga stores the
  token in OpenBao's key-value engine. No true OpenBao lease -- the
  token stays valid until Aga revokes it. The grant record's
  `openbao_lease_id` and `lease_expires_at` fields are `null`.
  If the Pasha is idle for days, nothing happens; the token remains
  valid. Rotation is Aga-driven (either on an explicit Sultan
  request or on a schedule Sultan configures).

- **Dynamic mode (Phase 2 path, GitHub App / DB creds / SSH CA):**
  Aga calls a dynamic secret engine; OpenBao mints a short-lived
  credential and issues a lease with TTL. Aga writes the lease ID
  and expiry into the grant. Janissary checks expiry before
  injecting (fails closed on expired). Aga renews before TTL while
  the province is running, stops renewing on destroy, and OpenBao
  revokes server-side at TTL -- so a missed cleanup is bounded by
  the lease window, not by Aga's reliability.

Phase 1 MVP ships KV mode. Phase 2 introduces GitHub App for repo
access (and so dynamic-mode grants) as soon as we have an operator
workflow for GitHub App install/rotate.

**Low-risk config** (Telegram bot tokens, public endpoints) -- Vizier
writes directly into containers. A leaked bot token lets someone chat
as the agent, not access code.

## CA Certificate Lifecycle

A Sultanate-wide CA certificate is generated once at deploy time. The CA
cert is installed in all province containers so mitmproxy can decrypt and
inspect HTTPS traffic. The CA private key is only accessible to the
Janissary container.

## Divan (Shared State)

Minimal shared state store: SQLite + FastAPI + a read-only Jinja2/HTMX
dashboard. Canonical contract for all components.

| Endpoint | Writer | Reader | Data |
|----------|--------|--------|------|
| `/provinces` | Vizier | Janissary, Aga, dashboard | ID, IP, name, status, firman, berat |
| `/grants` | Aga | Janissary, dashboard | Source IP + domain -> header injection, `openbao_lease_id`, `lease_expires_at` |
| `/whitelists` | Vizier (berat defaults), Aga (Sultan's changes) | Janissary, dashboard | Per-source allowed domains |
| `/blacklist` | Aga (on Sultan's instruction) | Janissary, dashboard | Global blocked domains |
| `/appeals` | Janissary | Kashif, Aga, Vizier, dashboard | Pending/resolved appeal records with Kashif verdict |
| `/port_requests` | Vizier (berat), Aga (decisions) | Aga, dashboard | Non-HTTP port access requests |
| `/audit` | Janissary, Aga, Kashif | dashboard | Decision log (requests, credential injections, screenings) |

**Access control:** Divan authenticates callers via pre-shared API keys (one
per component, generated at deploy time). Each key maps to a role with
endpoint-level read/write permissions. Grant secret values are only returned
to the Janissary role. See `DIVAN_API_SPEC.md` for details.

**Dashboard:** server-rendered (Jinja2 + HTMX) inside the same FastAPI
process. Eight pages: realm, province detail, province secrets, whitelist,
appeals, audit, blacklist, health. HTTP basic auth + `127.0.0.1` binding
on port 8601 (Sultan accesses via SSH tunnel). See `DIVAN_MVP_PRD.md`.

Divan ships with Janissary in the same repo. It starts before all other
components except OpenBao.

## Artifact Formats

**Firman** (container template) -- a directory containing a `firman.yaml`
manifest and optional supporting files. Stored at a convention path on the
host (`/opt/sultanate/firmans/<name>/`). Vizier resolves `--firman <name>`
by looking up this path. See `OPENCLAW_FIRMAN_MVP_PRD.md`.

**Berat** (agent profile) -- a directory containing a `berat.yaml` manifest
and template files (`SOUL.md`, `AGENTS.md`, `openclaw.json`). Stored at
`/opt/sultanate/berats/<name>/`. Vizier resolves `--berat <name>` by
looking up this path. Templates use `{{variable}}` syntax with simple string
substitution. See `OPENCLAW_CODING_BERAT_MVP_PRD.md`.

## GitHub Token Strategy

Sultan pre-creates GitHub PATs (fine-grained, scoped per repo), or Aga uses
a GitHub App to mint short-lived installation tokens via OpenBao's GitHub
plugin (if configured). Sultan gives PATs to Aga via Telegram: "Store this
token for repo X." Aga stores in OpenBao and writes the grant (with lease
ID) to Divan. Aga does not create long-lived tokens programmatically --
it stores and manages tokens that Sultan provides, or requests short-lived
tokens from GitHub App flow.

## Runtime

All in-province agents (Pashas) use the upstream `openclaw/openclaw` Docker
image. No custom Dockerfile. Each agent is differentiated by its
configuration (`SOUL.md`, `AGENTS.md`, `~/.openclaw/openclaw.json`, MCP
servers, env vars) applied at startup. Vizier and Aga also run as OpenClaw
agents, but outside of provinces (trusted host deployment).

See `OPENCLAW_FIRMAN_MVP_PRD.md`.

## Startup Order

OpenBao must start first (and unseal) -- Aga depends on it for credentials
to authenticate to everything else. Then Divan. Then Janissary + Kashif.
Then Aga. Then Vizier. If any earlier component is down, later ones fail
closed.

```
1. OpenBao (Secret Vault; Sultan manually unseals)
2. Divan (shared state + dashboard)
3. Janissary (proxy, reads from Divan)
4. Kashif (content inspector, loads three-layer models)
5. Aga (secrets, writes to Divan, direct networking; authenticates to
        OpenBao via AppRole)
6. Vizier (management, writes to Divan, through Janissary)
7. Provinces (on demand, through Janissary)
```

If Janissary is down, Vizier cannot reach Telegram -- Sultan loses the
management channel. This is fail-closed by design. Sultan has SSH to the
host as fallback (and the dashboard via SSH tunnel).

## Network Model

- Provinces on internal Docker network (`internal: true`, no external
  route)
- All province HTTP/HTTPS goes through Janissary (WireGuard transparent
  proxy)
- Non-HTTP access: blocked by default. Berat declares needed ports,
  Vizier writes them to Divan as requests, Aga asks Sultan for approval,
  then opens specific host:port pairs (iptables/Docker rules) and
  provisions service tokens. No auto-approve.
- Vizier's traffic goes through Janissary (same rules as provinces)
- Vizier has the Janissary security MCP tool -- can appeal blocks or ask
  Sultan to tell Aga to add permanent whitelist exceptions
- Aga has direct host networking (needs Telegram + local OpenBao only)
- OpenBao listener bound to `127.0.0.1`; Aga reaches it on localhost
- Kashif has no external networking; only receives HTTP posts from
  Janissary and Aga

## Traffic Rules (Janissary)

1. **Blacklist** -- domain on global blacklist? Block all traffic.
   Overrides whitelist.
2. **Whitelist** -- domain on province's allowlist? Pass all traffic.
3. **Non-whitelist read** -- GET/HEAD to unknown domain? Pass (browsing,
   docs).
4. **Non-whitelist write** -- POST/PUT/PATCH/DELETE to unknown domain?
   Block.
5. **Appeal** -- agent appeals a block with justification. Janissary
   forwards the payload + justification to Kashif `/screen/appeal`.
   Kashif returns `allow` (obvious safe), `block` (obvious bad), or
   `escalate` (unclear). On escalation the appeal flows to Aga and
   Sultan as before. Kashif fail-closed: timeout or down -> the appeal
   is held for manual review, never auto-approved.

## Appeal Flow

See `ARCHITECTURE.md` for the full timeline-style walkthrough. Summary:

1. Agent's write request to non-whitelisted domain -> blocked by
   Janissary.
2. Agent calls `appeal_request(url, method, justification)` via MCP
   tool.
3. Janissary writes the appeal to Divan and forwards payload +
   justification to Kashif `/screen/appeal`.
4. Kashif returns allow / block / escalate within ~2 s. Three branches:

   - **Kashif=allow** -> Divan auto-transitions appeal to approved;
     Janissary lets the retry through on next poll. **No Sultan
     notification** (obvious safe; audit entry only).
   - **Kashif=block** -> Divan auto-transitions to denied; retry still
     fails. **Both Sultan and Aga are notified** via an informational
     Telegram message. The decision is final, but the notification
     exists so the operator can spot drifting behaviour (3 blocks in
     10 min -> consider destroying the province).
   - **Kashif=escalate** (or Kashif unavailable) -> appeal stays
     pending; **both Sultan and Aga are notified** via actionable
     Telegram. Sultan decides approve/deny/whitelist/kill-province;
     Aga may add context from its own audit-history view.

5. Vizier is the Sultan-facing relay: it polls Divan audit and pending
   appeals and sends the Telegram messages. Aga polls the same records
   independently and contributes its own analysis to the Sultan chat.
6. All Kashif verdicts (including auto-approved) are written to
   `/audit`. Sultan can scan the dashboard's audit page at any time.

## Components

| Component | Repo | Description |
|-----------|------|-------------|
| **Vizier** | `vizier` | CLI + OpenClaw agent. Province lifecycle management. |
| **Janissary** | `janissary` | Transparent proxy (Sandcat fork) + credential injection. |
| **Kashif** | `janissary` | Local-LLM content inspector (3-layer CPU). Ships with Janissary. |
| **Divan** | `janissary` | SQLite + FastAPI + dashboard. Shared state store. Ships with Janissary. |
| **Aga** | `janissary` | Secret management OpenClaw agent. Ships with Janissary. |
| **OpenBao** | third-party | Local Secret Vault (pinned 2.5.3+, Apache-2.0, single binary). Aga is sole client. |
| **openclaw-firman** | `openclaw-firman` | Docker image + bootstrap for OpenClaw provinces. |
| **openclaw-coding-berat** | `openclaw-coding-berat` | Coding agent profile (soul, tools, whitelist, grants). |

## What's Deferred

- SentinelGate (MCP-level security) -- no tool-level RBAC
- Session tracking and behavioral analysis
- Multi-firman, multi-berat support (beyond the one coding profile)
- Cross-province coordination
- Dual audit sinks + signed audit records
- Signed-manifest cross-check between Janissary and Aga (compromised-
  Aga hardening)
- AppRole secret-id rotation on Aga restart
- Multi-operator Shamir split on OpenBao
- Multi-operator support
- Auto-unseal for OpenBao (manual unseal is Phase 1 default)
- GPU-class host for Llama Guard 3 8B or local Pasha LLMs (Phase 2 if
  throughput or privacy demands emerge)
