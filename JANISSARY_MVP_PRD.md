# PRD: Janissary MVP -- Egress Proxy for Sultanate

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).

## What Janissary Is

A dumb, deterministic transparent proxy (WireGuard). The only internet exit
for all provinces and Vizier. Based on a fork of Sandcat (VirtusLab,
https://github.com/VirtusLab/sandcat), using WireGuard transparent proxy
mode. Reads all state from Divan (shared state store). No LLM, no content
evaluation, no outbound access of its own.

Provinces must trust the Sultanate CA certificate for HTTPS interception to
work. The CA cert is generated at deploy time and distributed to all province
containers.

## Traffic Rules

Evaluated in order per request:

1. **Blacklist** -- domain on the global blacklist? Block all traffic
   (any method, including GET). Overrides whitelist.
2. **Whitelist** -- domain on the source's allowlist? Pass all traffic
   (any HTTP method).
3. **Read-only pass** -- GET or HEAD to a non-whitelisted domain? Pass.
   Agents can browse, read docs, download packages.
4. **Write block** -- POST, PUT, PATCH, DELETE to a non-whitelisted domain?
   Block. Return 403 with a message pointing the agent to the appeal tool.

Janissary does not inspect payloads, evaluate content, or make judgment
calls. It reads tables and applies them by source IP.

## Source Identification

Janissary identifies callers by source IP on the Docker internal network.
Each province container and Vizier have a known IP. Janissary reads the
mapping from config.

Per-source config:
- Whitelist (list of domains)
- Grant rules (credential injection)
- Blacklist is global (applies to all sources)

## Credential Injection

Janissary reads grants from Divan (written by Sentinel):

```json
{
  "source_ip": "172.18.0.5",
  "match": { "domain": "api.github.com" },
  "inject": { "header": "Authorization", "value": "Bearer ghp_xxxx" }
}
```

When a request matches source IP + domain:
1. Add or replace the specified header
2. Forward the request
3. No match? Forward without injection (still subject to traffic rules)

Grants are written to Divan by Sentinel. Divan's grants endpoint is
read-only for Janissary. Vizier and provinces have no access to grant
values.

## Appeal System

When a write request is blocked:

1. Agent calls `appeal_request(url, method, justification)` via MCP tool
   (which calls Janissary's HTTP API)
2. Janissary writes the appeal to Divan (`/appeals`)
3. Vizier polls Divan for pending appeals, relays to Sultan via Telegram
4. Sultan responds to Vizier: approve (one-time), whitelist (permanent),
   or deny
5. Vizier writes the decision to Divan
6. Janissary reads the decision from Divan and applies

Agents can also ask Sultan directly via Telegram to request a permanent
whitelist addition. Sultan tells Sentinel, Sentinel updates config.

### HTTP API

Janissary provides an HTTP API available to all provinces and Vizier.
MCP tools in provinces call this HTTP API:

```
appeal_request(
  url: string,
  method: string,
  justification: string
)
```

Returns: pending (appeal stored, waiting for Sultan's decision).

```
request_access(
  service: string,
  scope: string,
  justification: string
)
```

Routes to Sultan via Vizier. For new credentials, Sultan tells Sentinel.

Both tools are backed by Janissary's HTTP API. MCP tools in provinces wrap
these endpoints for agent convenience.

## HTTPS Handling

Janissary performs full MITM on all HTTPS traffic using the Sultanate CA
certificate (installed in all province containers). mitmproxy decrypts
every TLS connection, applies all 4 traffic rules (blacklist, whitelist,
read-only pass, write-block) uniformly to both HTTP and HTTPS, then
re-encrypts to the upstream server. Credential injection works on HTTPS
because the request is fully visible after decryption. Domains that break
under MITM (cert pinning) can be added to a passthrough list.

## Divan Integration

Janissary reads all its state from Divan's HTTP API:

| Divan Endpoint | Janissary Uses For |
|----------------|-------------------|
| `GET /provinces` | Source IP -> province mapping |
| `GET /whitelists/{source}` | Per-source allowed domains |
| `GET /grants/{source}` | Credential injection rules |
| `GET /blacklist` | Global blocked domains |
| `POST /appeals` | Store new appeals |
| `GET /appeals?status=resolved` | Read Sultan's decisions |

Janissary polls Divan periodically (configurable, default 5s) and caches
locally. If Divan is unreachable, Janissary enforces last-cached rules.
If Janissary has never successfully read from Divan (fresh start), it
blocks all traffic.

## Non-HTTP Traffic

Janissary is HTTP/HTTPS only. Non-HTTP traffic (SSH, database, etc.) is
handled at the Docker network level by Sentinel:

- Provinces are on `internal: true` network (no external route)
- Berat declares non-HTTP needs (e.g., `github.com:22` for SSH)
- Vizier writes these declarations to Divan as requests
- Sentinel reads them, asks Sultan for approval via Telegram
- On approval, Sentinel opens specific host:port pairs via iptables/Docker
  network rules (requires root) and provisions service tokens
- No auto-approve -- every non-HTTP port opening requires Sultan's decision
- Vizier has no network-level privileges -- it only creates containers on
  the internal network

## What Janissary Does NOT Do

- No LLM, no content evaluation
- No outbound access of its own (only forwards)
- No secret management (Sentinel's job)
- No province management (Vizier's job)
- No automated triage (deferred: Kashif)
- No session tracking (deferred: SentinelGate)

## Phase 1 Scope

**In scope:**
- HTTP/HTTPS transparent proxy (WireGuard) with CONNECT tunnel support
- Per-source whitelists (pass all traffic to listed domains)
- Read-only pass for non-whitelisted domains (GET/HEAD only)
- Write block for non-whitelisted domains (POST/PUT/PATCH/DELETE)
- Global blacklist (block all methods)
- Credential injection from Divan grants
- HTTP API for appeals and access requests (called by MCP tools in provinces)
- Appeal storage and decision polling via Divan
- Divan polling with local cache, fail-closed
- Audit log of all traffic decisions

**Deferred:**
- Size gate (payload-based blocking)
- Kashif integration (LLM content triage)
- Automated appeal approval
- Session-aware policies
- SentinelGate integration (tool-level RBAC)
- Non-HTTP protocol proxying
