# PRD: Janissary v2 -- Security Gateway for Sultanate

> For shared glossary, deployment model, and component overview see
> [SULTANATE.md](SULTANATE.md).

## Vision

Janissary is the security perimeter for Sultanate. It is a dumb, deterministic
egress proxy running on the host machine as root. No province (isolated
container) can reach the internet except through Janissary.

Janissary does not know what a province is. It reads rules from Divan (the
shared state store) and applies them based on source IP. Kashif (the content
inspector) screens appeals and Aga ingress for malice. Aga (the
security advisor) -- a trusted Hermes agent also running as root -- is the
intelligence layer: it manages secrets, contextualizes alerts, and curates
blacklists. All Aga inputs are pre-screened by Kashif.

Janissary, Kashif, and Aga are one product, one repo.

## Product Boundary

**Janissary provides:**
- Egress proxy -- the only internet exit for all province containers
- Traffic filtering (whitelist, size gate on outbound payloads, blacklist)
- Transparent credential injection at the proxy level
- HTTP/HTTPS with CONNECT tunnel support (WebSocket compatible)
- Audit trail of all traffic decisions

**Kashif provides:**
- Content inspection -- screens all content for malice before delivery
- Layer 4 appeal triage (approve obvious safe, block obvious bad, escalate
  unclear to Aga)
- Aga ingress screening -- all Pasha (agent inside province) originated
  content and fetched
  web pages screened before Aga ingests them
- Prompt injection and manipulation detection
- Fail-closed behavior -- if down or unsure, block and alert Sultan
  (human operator)

**Aga provides:**
- Secret management (creation, rotation, revocation, provisioning)
- Operator-facing alert summaries with context for Sultan
- Blacklist curation
- Access request review (escalations from Kashif)
- Approval context preparation for Sultan
- Audit queries and policy explanations

**Divan provides (shared state, used by both):**
- Province registry (ID, IP, status, firman -- written by Vizier)
- Grant table (source IP + destination -> credential injection rule)
- Blacklist
- Whitelist per source (province allowlists, Aga's own whitelist)
- Audit log
- Web dashboard (read-only for Sultan)

**Janissary + Aga do NOT provide:**
- Container orchestration (Vizier's job)
- Agent runtime or task management (runtime's job)
- Province lifecycle management (Vizier's job)
- Code review or task quality assessment

## Design Constraints

- **Janissary is dumb.** It reads tables from Divan and applies rules. No
  LLM, no content evaluation, no state management, no agent communication,
  no province awareness.
- **Janissary has no outbound access.** Janissary itself never initiates
  outbound internet connections. It only forwards province/Aga traffic
  and communicates with Divan and OpenBao (both local). This prevents
  a compromised Janissary from becoming an open relay.
- **Deterministic enforcement.** Traffic filtering and credential injection
  never depend on an LLM call. Whitelists, blacklists, size thresholds, and
  grant tables are the enforcement layer.
- **Kashif is paranoid, not smart.** Kashif screens content for malice but
  cannot approve access or modify policy. It triages appeals (approve obvious
  safe, block obvious bad, escalate unclear). Fail-closed: if Kashif is down
  or unsure, block and alert Sultan. High false-positive rate is acceptable;
  high false-negative rate is not.
- **LLM is advisory only.** Kashif and Aga review appealed traffic.
  They can escalate to Sultan but cannot override a block or grant access on
  their own without Sultan's approval (unless within auto-approve policy).
- **Network-level enforcement.** Agents cannot bypass Janissary because it is
  the network route, not a tool they choose to call. The only application-level
  integration is the security MCP tool for appeals.
- **HTTPS only (Phase 1).** Only HTTP/HTTPS traffic (with CONNECT tunnel
  support for WebSocket) passes through the proxy. Non-HTTP protocols (SSH,
  raw TCP, gRPC) are blocked by the network topology. Support for additional
  protocols (database TCP, VPN tunnels) is a future consideration.
- **Aga is constrained.** Aga's own outbound traffic goes through
  Janissary with a strict whitelist-only policy (size gate = 0, no default
  pass). Aga cannot expand its own whitelist -- only Sultan can.
- **Minimal trust in agents.** Assume any agent could be confused, misled, or
  adversarial. Janissary enforces policy regardless of agent intent.
- **Province-scoped grants.** A grant for one source IP does not apply to
  another. Revocation is per-province.
- **One-way dependencies.** Janissary reads from Divan, never calls Vizier
  (deployment orchestrator) or Aga directly. Aga reads/writes
  Divan, never calls Vizier. Vizier writes to Divan, never calls Janissary.

## Traffic Layers

All outbound HTTP/HTTPS requests pass through Janissary. Non-HTTP protocols
are blocked by the Docker network topology (provinces can only reach
Janissary's proxy port).

Janissary applies different rules depending on the traffic source:

### Province Traffic

Evaluated in this order:

**Layer 1: Whitelist (pass silently)**
Domain is on the source's allowlist in Divan. Traffic goes through, no
inspection.

**Layer 2: Size Gate (catch code leaks)**
Outbound request payload to a non-whitelisted domain:
- Small outbound payload (< 5KB) -- pass. Normal browsing, small API calls.
- Large outbound payload (>= 5KB) -- blocked. Escalated to Kashif for review.

Size gate applies to outbound request payloads only. Inbound responses are
not size-gated -- downloading packages, cloning repos, and fetching
documentation proceed regardless of response size.

The 5KB threshold is configurable per province and is a conservative starting
point. To be tuned after observing real traffic patterns.

**Layer 3: Blacklist (block known bad)**
Curated by Aga in Divan. Known paste sites, exfiltration endpoints,
malicious domains. Blocked immediately.

**Layer 4: Appeal (agent requests review)**
When an agent's request is blocked (by Layer 2 or 3) and the agent believes
it is legitimate:

1. Agent calls the security MCP tool:
   `appeal_request(url, method, full_payload, justification)`
2. Kashif (content inspector) reviews the full payload (fast, paranoid):
   - Obviously safe -- approve
   - Obviously bad -- block
   - Unclear -- escalate to Aga
   - Payload too large for Kashif's context window -- escalate to Aga
3. Aga reviews with broader context, can approve or escalate to Sultan
4. Sultan makes final decision if needed

**Default:** Non-whitelisted, small payload -- pass.

### Aga Traffic

Strict whitelist only. No size gate, no default pass. Everything not
whitelisted is blocked. Aga has root access and can appeal to Sultan
to expand its whitelist, but cannot modify it itself.

## Credential Injection

Agents never hold dangerous secrets. Credentials are injected transparently
by Janissary at the proxy level.

### How It Works

Janissary reads the **grant table** from Divan:

```json
{
  "grant_id": "g-abc-123",
  "source_ip": "172.18.0.5",
  "match": {
    "domain": "api.github.com",
    "path_prefix": "/repos/stranma/EFM"
  },
  "inject": {
    "header": "Authorization",
    "value": "Bearer ghp_xxxxxxxxxxxx"
  },
  "openbao_lease_id": "auth/token/create/abcd1234",
  "lease_expires_at": "2026-04-23T18:00:00Z"
}
```

The `value` field holds the current credential. Aga renews the OpenBao
lease before expiry and rewrites `value` + `lease_expires_at` in place.
Janissary reads the record as-is -- it does not call OpenBao directly.
If Aga fails to renew before `lease_expires_at`, Janissary treats the
grant as expired and stops injecting (fail-closed); the credential also
expires server-side in OpenBao at the same time.

**Phase 2 option:** move `value` out of Divan entirely. Janissary fetches
the current credential from OpenBao on demand (with short in-memory cache)
using its own read-only AppRole. This keeps Divan free of raw secrets and
limits credential-in-memory to the Janissary process only. Deferred for
Phase 1 simplicity.

When Janissary sees an outbound request:

1. **Identify source** -- by source IP on the internal Docker network
2. **Match against grant table** -- domain + optional path prefix
3. **Inject** -- add/replace the specified header(s) with the grant's value
4. **No match** -- request passes through without injection (still subject
   to traffic layers)

### Two Classes of Secrets

- **Dangerous secrets** (repo access, API keys, cloud credentials) -- Aga
  manages, Janissary injects transparently via grant table. Agent never sees
  the raw value.
- **Non-sensitive config** (public endpoints, feature flags, etc.) -- passed
  directly to province environment. Out of scope for Janissary/Aga.

Sultan decides which class a secret belongs to.

### Grant Lifecycle

```text
1. Province created from firman (container template)
   --> Vizier writes province to Divan (ID, IP, status, firman)
   --> firman defines default grants
2. Aga reads new province from Divan, provisions credentials
   --> calls OpenBao to generate credential (dynamic where possible:
       GitHub App token, DB creds, SSH cert, PKI cert)
   --> OpenBao returns a value and a lease with TTL
   --> Aga writes grant rules to Divan's grant table, tagged
       with the OpenBao lease ID
3. Pasha (agent inside province) works, git/API calls go through Janissary transparently
   --> Janissary reads grant table, injects credentials on match
4. Pasha needs new access mid-task
   --> Pasha requests via MCP tool or runtime channel to Aga
5. Aga evaluates:
   --> auto-approve if within policy (e.g., read access to firman
       default repos, standard package registries)
   --> escalate to Sultan if outside policy (e.g., write access,
       new API keys, unknown services, any request after a
       security incident)
6. Approved --> Aga calls OpenBao, writes lease-bound grant to Divan
7. Province destroyed --> Vizier updates Divan, Aga revokes
   all OpenBao leases for that province. Any missed revocation is
   bounded by the lease TTL -- OpenBao expires the credential
   server-side even if Aga fails to act.
```

**Why OpenBao (not a static KV vault like Infisical):**
- **Dynamic secrets** -- GitHub App tokens, DB creds, SSH CA-issued keys,
  PKI-issued certs generated per-province on demand. Short-lived by default.
- **Lease-based revocation** -- credentials expire server-side on TTL.
  Trust in Aga to catch every province destroy event becomes a
  latency optimization, not a correctness requirement.
- **Audit devices** -- every secret access is logged with HMAC integrity;
  Divan reads this feed for the dashboard.
- **Transit engine** -- Divan can encrypt audit records at rest via
  OpenBao without holding key material.
- **Apache 2.0, single binary, self-hosted.** No vendor lock-in.

Aga is the sole OpenBao client. Pashas never authenticate to OpenBao;
they only ever see Janissary-injected headers.

**Secret zero:** Aga needs credentials to reach OpenBao. Phase 1
default is manual unseal by Sultan at boot (Shamir split not required for
single-operator). Auto-unseal against a host-KMS or file-wrapped key is
available for dev; document the trade-off explicitly -- auto-unseal means
Aga-compromise equals OpenBao-compromise.

Sultan can revoke any grant at any time via Aga.

## Security MCP Tool

Janissary provides an MCP server available in every province. This is the only
application-level integration point -- everything else is network-level.

**Primary use: Layer 4 appeals.**

```
appeal_request(
  url: string,
  method: string,
  payload: string,       # full content
  justification: string  # why the agent needs this
)
```

Returns: approved (one-time or added to whitelist) or denied with reason.

**Secondary use: requesting new access.**

```
request_access(
  service: string,       # e.g. "github", "stripe"
  scope: string,         # e.g. "stranma/EFM write"
  justification: string
)
```

Routed to Aga for evaluation.

For non-MCP runtimes, both tools are backed by Janissary's HTTP API as a
fallback.

## Kashif (Content Inspector)

Kashif is a paranoid content inspector running a local LLM on the host. It
has one job: "can this be in any way malicious?"

**Responsibilities:**

- **Appeal triage** -- reviews Layer 4 appeals. Receives full payload, URL,
  method, headers, and justification. Approves obvious safe, blocks obvious
  bad, escalates unclear to Aga.
- **Aga ingress screening** -- all Pasha-originated content (appeal
  justifications, access request text, freeform input) is screened by Kashif
  before reaching Aga's context window.
- **Fetched content inspection** -- when Aga fetches web content (after
  Sultan approves the whitelist addition), Kashif inspects the fetched content
  before Aga ingests it.
- **Prompt injection detection** -- flags manipulation attempts, injection
  patterns, and unexpected payloads.

**Behavior:**

- Paranoid by default -- when in doubt, block and alert Sultan
- Must respond within a configurable timeout (e.g., 5s). If no response,
  default to block.
- High false-positive rate is acceptable; high false-negative rate is not.
- Fail-closed: if Kashif is down, all appeals are blocked and all Aga
  ingress is held until Kashif recovers or Sultan intervenes.

Kashif is NOT in the hot path for normal traffic. It processes appeals and
screens Aga inputs. Whitelisted traffic passes through Janissary without
touching Kashif.

## Aga Agent

Aga is a trusted Hermes agent running as root on the host. It is the
operator-facing intelligence layer.

**Aga is a trusted component.** It runs with elevated privileges and has
real authority over secrets and access grants. Unlike Pashas (agents inside
provinces), Aga is not sandboxed -- it is part of the system's trusted
core alongside Vizier and Janissary. Trust is layered: all external and
Pasha-originated content reaching Aga is first screened by Kashif for
prompt injection and manipulation attempts.

**Responsibilities:**
- **Secret management** -- creates tokens, rotates them, provisions grants,
  revokes on province destruction. Asks Sultan for approval when needed.
- **Alert contextualization** -- every alert passes through Aga before
  reaching Sultan. Adds explanation of what happened, why it was blocked,
  what the options are.
- **Blacklist curation** -- maintains and updates the blacklist in Divan
  based on observed traffic patterns and known bad destinations.
- **Access request review** -- reviews escalations from Kashif with broader
  context. Can approve, deny, or escalate to Sultan.
- **Approval context** -- prepares grant/access requests with context so
  Sultan can make informed decisions.
- **Audit queries** -- answers Sultan's questions about province activity.

**Aga is not optional.** It ships with Janissary and Kashif in Phase 1.

**Aga's own security:**
- All Aga inputs are pre-screened by Kashif for malice
- Aga's outbound traffic goes through Janissary with whitelist-only
  policy (no size gate pass-through)
- Any web content Aga fetches is inspected by Kashif before ingestion
- Aga cannot expand its own whitelist -- only Sultan can
- Aga can appeal to Sultan to expand its whitelist
- Sultan can modify Aga's whitelist directly (root access)

**Example alerts Aga sends to Sultan:**
- "Province (container) A tried to POST 45KB to paste.mozilla.org -- blocked
  by size gate. Kashif flagged the payload as a Python module. Approve or
  deny?"
- "Province B is requesting write access to stranma/EFM because it needs
  to push a dependency update. Firman default is read-only. Approve?"
- "Province C hit the blacklist 3 times in 10 minutes targeting different
  paste sites. Possible exfiltration attempt. Kill province?"

## Divan (Shared State)

Divan is the shared state store and API for the Sultanate. It is not an
orchestrator -- components report state to it, others read from it.

**What Divan holds:**
- Province registry (ID, IP, status, firman used)
- Grant table (source IP + destination -> credential injection rule)
- Blacklist (curated by Aga)
- Whitelist per source (province allowlists, Aga's own whitelist)
- Audit log

**Who touches Divan:**

| Component | Reads | Writes |
|-----------|-------|--------|
| Vizier | -- | Province registry (creates/updates status) |
| Aga | Province registry, grants, audit | Grants, blacklist, audit |
| Janissary | Grant table, blacklist, whitelists | Audit log |
| Web dashboard | Everything | Nothing (read-only) |

**Web dashboard:** simple read-only view for Sultan. Shows realm status,
active provinces, grants, recent audit entries, pending approvals.

**Implementation:** SQLite + lightweight HTTP API (Phase 1). Can evolve to
Postgres if needed.

## Metrics

Janissary tracks and exposes via Divan (queryable by Aga and web
dashboard):

- Total requests per source IP (passed/blocked/appealed)
- Active grants per province
- Escalation count and Aga/Sultan response time
- Blacklist hit count (which domains triggered)
- Appeal outcomes (approved/denied/escalated)
- Blocked payload sizes (for tuning size gate threshold)

## Phase 1 Scope

**In scope:**
- Janissary HTTP/HTTPS egress proxy with CONNECT tunnel support
- Traffic layers: whitelist, size gate (5KB default, outbound payloads only),
  blacklist, appeal (routed to Kashif)
- Security MCP tool for appeals and access requests
- Kashif content inspector for appeal triage and Aga ingress screening
- Aga agent (non-optional), all inputs screened by Kashif
- Transparent credential injection via grant table (lease-bound values)
- Secret management by Aga via OpenBao (create, rotate, revoke,
  lease renewal). OpenBao deployed as single local binary, bound to
  127.0.0.1, manual unseal at boot.
- Grant lifecycle tied to province lifecycle via Divan
- Divan shared state store with API
- Web dashboard (read-only realm status for Sultan)
- Audit logging and metrics
- All alerts contextualized by Aga before reaching Sultan
- Aga's own traffic constrained to whitelist-only, fetched content
  inspected by Kashif
- Janissary has no outbound access of its own, no LLM
- All components fail closed

**Deferred:**
- Non-HTTP protocol support (database TCP, VPN tunnels, gRPC)
- Multi-machine deployment
- Automatic pattern learning from approved/denied history
- Advanced anomaly detection (frequency analysis, first-seen domain
  tracking, unusual endpoint detection on whitelisted domains)
- Grant TTL and automatic expiry
- Multi-Sultan support (multiple operators)
