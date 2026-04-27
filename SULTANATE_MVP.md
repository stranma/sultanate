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
                       Gateway daemon on port 18789 (loopback only).
        Pasha <-> Sultan via dedicated per-province Telegram bot
                  (one bot token per Pasha; provisioned by Vizier
                   from a pre-allocated pool, returned on destroy).
```

## Communication Channels

Sultan talks to each agent via a dedicated Telegram bot. There are
**2 + N bots** in MVP: one for Vizier, one for Aga, plus one per
active province.

| Bot | Speaks for | Lifecycle |
|-----|-----------|-----------|
| Vizier bot | Vizier (province lifecycle, escalated-appeal relay, Kashif-block notifications) | Long-lived; one per Sultanate deployment |
| Aga bot | Aga (security alerts, port-request approvals, GitHub App bootstrap) | Long-lived; one per Sultanate deployment |
| Pasha bot (one per province) | The Pasha inside that province (task chat, status, clarifications) | Acquired from a pre-allocated bot pool when the province is created; returned to the pool on destroy |

Each bot's `channels.telegram.allowFrom` in OpenClaw config is set to
Sultan's Telegram user ID, so only Sultan can reach the bot.

**Trust property -- per-province blast radius.** A Pasha's bot token is
scoped to that one province. Compromise of one Pasha's token (e.g., a
rogue Pasha exfiltrates its own token, or a token leaks from the pool's
storage):

- lets an attacker chat *as that Pasha to Sultan* and read whatever
  context Sultan typed in that thread
- **cannot reach** any other Pasha's bot, Vizier's bot, or Aga's bot
- **cannot read or write secrets** (those live in OpenBao, behind
  Aga; the Pasha never had them in the first place)
- **cannot bypass Janissary** -- still subject to the same WireGuard
  + iptables kill-switch and traffic rules
- **cannot mint, rotate, or read GitHub App tokens** (Aga's job, Aga's
  OpenBao AppRole)

Blast radius is bounded to that single province's chat thread. Sultan
notices the impersonation by the Pasha behaving oddly; the response is
to destroy the province via Vizier (which returns the bot to the pool
for re-issue with a fresh token, after Sultan revokes the old one in
BotFather).

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

**Dangerous secrets** (GitHub tokens, API keys) -- Aga creates them,
stores them in OpenBao, and writes a grant record to Divan. Janissary
reads grants from Divan and injects into request headers at the proxy
level. Containers never see these values.

**Sultan does not paste tokens.** Sultan's role is one-time setup:
install a GitHub App on the repos Sultanate should manage, and hand
the App private key to Aga. From then on, Aga mints short-lived
installation tokens per-province on demand. (For services without
automation, Sultan can still paste tokens in Telegram -- see KV
fallback below -- but GitHub is fully automated.)

### Grant modes

- **Dynamic mode (Phase 1 default, GitHub App):**
  Aga holds the GitHub App private key in OpenBao KV. When a province
  is created (or needs rotation), Aga mints a GitHub App installation
  access token scoped to the province's repo, with GitHub's
  hard-capped TTL of 1 hour. Aga writes the grant to Divan with an
  Aga-generated lease ID (`github-app:prov-XXXXXX`) and the
  `lease_expires_at` returned by GitHub. A background renewal loop in
  Aga refreshes every ~30 min while the province is running, stops
  refreshing on destroy, and GitHub kills the token within 1 hour
  naturally. Janissary checks expiry before injecting and fails
  closed on expired (audit entry, `severity=alert`).

- **KV fallback (edge cases):**
  For services that do not have a dynamic mint path and where Sultan
  manually pastes a token in Telegram (e.g., a third-party API Aga
  cannot automate against), Aga stores the token in OpenBao KV. The
  grant's `openbao_lease_id` and `lease_expires_at` are `null`;
  Janissary injects unconditionally; the token lives until Sultan
  tells Aga to revoke. This is a fallback, not the default.

Phase 2 may add a dedicated OpenBao secret engine plugin for GitHub
Apps (community `vault-plugin-secrets-github` may work directly);
for MVP the minting logic lives in Aga itself, keyed off the App
private key held in OpenBao KV.

**Low-risk config** (Telegram bot tokens, public endpoints) -- Vizier
writes directly into containers. A leaked Pasha bot token has a
per-province blast radius (see Communication Channels above). A
leaked Vizier or Aga bot token lets an attacker impersonate Sultan
to that one agent, but they still cannot bypass Kashif on Aga
ingress, cannot bypass Janissary, and cannot read OpenBao secrets.

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

**Sultan's one-time setup (outside Sultanate):**

1. Create a Sultanate GitHub App in Sultan's GitHub account/org.
2. Configure the App with minimal repo permissions (`contents:write`,
   `pull_requests:write`, `metadata:read`; add more per-repo as
   needed later via the App settings UI).
3. Install the App on the repos Sultanate should manage. New repos
   get added to the installation later by Sultan in the GitHub UI;
   no Sultanate-side action required at install time.
4. Download the App private key (PEM) once; hand it to Aga by
   dropping the file into `/opt/sultanate/bootstrap/github-app.pem`
   during first boot (deploy-script prompts for it), OR send it
   via Telegram once for Aga to persist into OpenBao KV.

**Per-province provisioning (automatic, Aga-driven):**

1. Vizier creates province `prov-a1b2c3` for repo `stranma/EFM`.
   Vizier writes province record to Divan.
2. Aga sees the new province. Reads the GitHub App private key from
   OpenBao KV (`kv/github-app/private-key`).
3. Aga generates a JWT signed with the App private key, calls
   `POST /app/installations/{installation_id}/access_tokens` scoped
   to `stranma/EFM`. GitHub returns a token with 1-hour TTL.
4. Aga writes grant to Divan:
   ```json
   {
     "province_id": "prov-a1b2c3",
     "source_ip": "10.13.13.5",
     "match":  { "domain": "api.github.com" },
     "inject": { "header": "Authorization", "value": "<token>" },
     "openbao_lease_id":  "github-app:prov-a1b2c3",
     "lease_expires_at":  "<token-expiry from GitHub>"
   }
   ```
5. Aga's renewal loop (runs every ~15 min) refreshes any grant
   where `lease_expires_at < now + 20 min`, updating `inject.value`
   and `lease_expires_at` in place via `PATCH /grants/{id}`. Stops
   refreshing when the province status becomes `stopped` or
   `destroying`.

**Sultan's day-to-day interaction with GitHub credentials:** none.
Aga handles minting, renewal, and revocation silently. Sultan sees
grant entries on the dashboard but never pastes or handles tokens.

**Non-GitHub services** (if a Pasha needs cloud API credentials,
third-party service tokens) fall back to KV mode: Sultan provides the
token once via Telegram, Aga stores it in OpenBao KV, grant has no
lease, lives until Sultan says to revoke. Phase 2 expands dynamic
minting to more services as appropriate.

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
