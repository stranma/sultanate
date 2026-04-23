# PRD: Janissary MVP -- Egress Proxy for Sultanate

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For detailed implementation contract see [JANISSARY_SPEC.md](JANISSARY_SPEC.md).
> For the content-inspector sibling see [KASHIF_MVP_PRD.md](KASHIF_MVP_PRD.md).

## What Janissary Is

A dumb, deterministic transparent proxy (WireGuard). The only internet exit
for all provinces and Vizier. Based on a fork of Sandcat (VirtusLab,
https://github.com/VirtusLab/sandcat), using WireGuard transparent proxy
mode. Reads all state from Divan (shared state store). No LLM, no content
evaluation, no outbound access of its own. Kashif (the content inspector,
ships alongside Janissary) handles any LLM-based judgement.

Provinces must trust the Sultanate CA certificate for HTTPS interception
to work. The CA cert is generated at deploy time and distributed to all
province containers.

## Traffic Rules

Evaluated in order per request:

1. **Blacklist** -- domain on the global blacklist? Block all traffic
   (any method, including GET). Overrides whitelist.
2. **Whitelist** -- domain on the source's allowlist? Pass all traffic
   (any HTTP method).
3. **Read-only pass** -- GET or HEAD to a non-whitelisted domain? Pass.
   Agents can browse, read docs, download packages.
4. **Write block** -- POST, PUT, PATCH, DELETE to a non-whitelisted
   domain? Block. Return 403 with a message pointing the agent to the
   appeal tool.

Janissary does not inspect payloads, evaluate content, or make judgment
calls. It reads tables and applies them by source IP.

## Source Identification

Janissary identifies callers by source IP on the WireGuard subnet
(`10.13.13.0/24`). Each province container and Vizier have a known peer
IP. Janissary reads the mapping from Divan (`/provinces`).

Per-source config:
- Whitelist (list of domains)
- Grant rules (credential injection)
- Blacklist is global (applies to all sources)

## Credential Injection

Janissary reads grants from Divan (written by Aga):

```json
{
  "source_ip": "10.13.13.5",
  "match":  { "domain": "api.github.com" },
  "inject": { "header": "Authorization", "value": "Bearer ghp_xxxx" },
  "openbao_lease_id":  "auth/token/create/xyz789",
  "lease_expires_at":  "2026-04-24T10:30:00Z"
}
```

When a request matches source IP + domain:

1. Check `lease_expires_at`. If set and past current UTC time, **skip
   injection** and write an audit entry with `severity=alert,
   action=lease_expired, credential_injected=false`. The request
   proceeds without the header; upstream will return `401`/`403`
   naturally, signalling to the agent that the token is stale. Aga
   sees the audit entry and re-issues or renews.
2. Otherwise, add or replace the specified header with `inject.value`.
3. Forward the request.
4. Write audit entry with `credential_injected=true, severity=info`.

Grants with `openbao_lease_id=null` and `lease_expires_at=null` (KV-
stored PATs in Phase 1 MVP) inject unconditionally -- no expiry check
applies. See `SULTANATE_MVP.md` credential model for KV vs dynamic mode.

Grants are written to Divan by Aga. Divan's grants endpoint is
read-only for Janissary. Vizier and provinces have no access to grant
values.

## Appeal System

When a write request is blocked:

1. Agent calls `appeal_request(url, method, payload, justification)`
   via the MCP tool (which calls Janissary's HTTP API).
2. Janissary does two writes in parallel:
   - `POST /appeals` to Divan (creates the appeal record, status
     `pending`, `kashif_verdict=null`).
   - `POST /screen/appeal` to Kashif (forwards the payload +
     justification for triage).
3. Kashif runs its three-layer pipeline (LLM Guard regex, Prompt
   Guard 2 22M, Llama Guard 3 1B Q4) and returns `allow` / `block` /
   `escalate` within ~2 s. Kashif writes the verdict back to Divan
   via `PATCH /appeals/{id}/kashif_verdict`.
4. Divan auto-transitions on `allow` (approved, one-time) and on
   `block` (denied). On `escalate` (or Kashif timeout / unavailable),
   the appeal stays pending and Vizier relays to Sultan via Telegram.
5. Both Sultan and Aga are notified for `block` (informational, so
   they can watch for Pasha drift) and `escalate` (actionable, Sultan
   decides). Only `allow` stays silent.
6. Janissary reads the decision from Divan on its next poll (default
   5 s) and applies it on retry.

See `ARCHITECTURE.md` appeal flow for the full timeline.

### HTTP API

Janissary provides an HTTP API available to all provinces and Vizier.
MCP tools in provinces call this HTTP API:

```
appeal_request(
  url: string,
  method: string,
  payload: string,        # full request body, forwarded to Kashif
  justification: string
)
```

Returns: pending (appeal stored, being screened by Kashif; decision
follows in seconds for allow/block or after Sultan reply for
escalate).

```
request_access(
  service: string,
  scope: string,
  justification: string
)
```

Routes to Sultan via Vizier for new credentials. Text is also
screened by Kashif (`/screen/ingress`) before reaching Aga's LLM
context.

Both tools are backed by Janissary's HTTP API. MCP tools in provinces
wrap these endpoints for agent convenience.

## HTTPS Handling

Janissary performs full MITM on all HTTPS traffic using the Sultanate
CA certificate (installed in all province containers). mitmproxy
decrypts every TLS connection, applies all 4 traffic rules uniformly
to both HTTP and HTTPS, then re-encrypts to the upstream server.
Credential injection works on HTTPS because the request is fully
visible after decryption. Domains that break under MITM (cert
pinning) can be added to a passthrough list.

## Divan Integration

Janissary reads all its state from Divan's HTTP API:

| Divan Endpoint | Janissary Uses For |
|----------------|-------------------|
| `GET /janissary/state` | Bulk snapshot (provinces, grants, whitelists, blacklist, approved_appeals) on every poll |
| `POST /appeals` | Store new appeals |
| `PATCH /appeals/{id}/kashif_verdict` | (Janissary writes `escalate` only when Kashif times out; normal writes come from Kashif itself) |
| `POST /audit` | Append audit entries |

Janissary polls Divan every 5 s (configurable) and caches locally. If
Divan is unreachable, Janissary enforces last-cached rules. If
Janissary has never successfully read from Divan (fresh start), it
blocks all traffic (fail-closed).

## Non-HTTP Traffic

Janissary is HTTP/HTTPS only. Non-HTTP traffic (SSH, database, etc.)
is handled at the Docker network level by Aga:

- Provinces are on `internal: true` network (no external route)
- Berat declares non-HTTP needs (e.g., `github.com:22` for SSH)
- Vizier writes these declarations to Divan as port_requests
- Aga reads them, asks Sultan for approval via Telegram
- On approval, Aga opens specific host:port pairs via iptables/Docker
  network rules (requires root) and provisions service tokens
- No auto-approve -- every non-HTTP port opening requires Sultan's
  decision
- Vizier has no network-level privileges -- it only creates containers
  on the internal network

## What Janissary Does NOT Do

- No LLM, no content evaluation (Kashif's job)
- No outbound access of its own (only forwards)
- No secret management (Aga's job)
- No province management (Vizier's job)
- No session tracking (deferred: SentinelGate)

## Phase 1 Scope

**In scope:**
- HTTP/HTTPS transparent proxy (WireGuard) with CONNECT tunnel support
- Per-source whitelists (pass all traffic to listed domains)
- Read-only pass for non-whitelisted domains (GET/HEAD only)
- Write block for non-whitelisted domains (POST/PUT/PATCH/DELETE)
- Global blacklist (block all methods)
- Credential injection from Divan grants, lease-aware (skip on
  expired lease)
- HTTP API for appeals and access requests (called by MCP tools in
  provinces)
- Appeal storage, Kashif forwarding, and decision polling via Divan
- Divan polling with local cache, fail-closed
- Audit log of all traffic decisions to Divan (`/audit`)

**Deferred:**
- Size gate (payload-based blocking)
- Automated appeal approval beyond Kashif's three-layer pipeline
- Session-aware policies
- SentinelGate integration (tool-level RBAC)
- Non-HTTP protocol proxying
