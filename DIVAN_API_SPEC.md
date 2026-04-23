# Divan API Specification

> Technical contract for the shared state store. All components depend on
> this interface. See [SULTANATE_MVP.md](SULTANATE_MVP.md) for context
> and [DIVAN_MVP_PRD.md](DIVAN_MVP_PRD.md) for scope.

## Overview

SQLite database + Python HTTP API (FastAPI). The JSON API listens on
`0.0.0.0:8600` (reachable from other Sultanate containers on the
internal Docker network). A server-rendered Jinja2/HTMX dashboard lives
in the same process on `127.0.0.1:8601` (host-localhost only; Sultan
reaches it via SSH tunnel). All component communication is JSON over
HTTP. No TLS (trusted local network only).

## Authentication

Pre-shared API keys, one per component. Passed as
`Authorization: Bearer <key>`. Keys are generated at deploy time and
stored in `/opt/sultanate/divan.env`.

Each key maps to a role:

| Role | Key env var | Permissions |
|------|-------------|-------------|
| `vizier` | `DIVAN_KEY_VIZIER` | Read/write provinces, write whitelists (berat defaults), read appeals, write appeal decisions, write port_requests, write audit |
| `aga` | `DIVAN_KEY_AGA` | Read provinces, read/write grants (including secret values), read/write whitelists, read/write blacklist, read/write appeal decisions, read/write port_requests, write audit |
| `janissary` | `DIVAN_KEY_JANISSARY` | Read provinces, read grants (including secret values), read whitelists, read blacklist, write appeals, read appeal decisions, write audit |
| `kashif` | `DIVAN_KEY_KASHIF` | Read appeals, write appeal `kashif_verdict`, write audit |
| `dashboard` | `DIVAN_KEY_DASHBOARD` | Read everything (grant `inject.value` is always masked for this role); no writes |

Grant secret values (`inject.value`) are only returned in plaintext for
`aga` and `janissary` roles. Other roles see `inject.value: "***"` in
responses.

Unauthenticated requests return `401`. Unauthorized access returns `403`.

## Common Response Format

**Success:**
```json
{ "data": <result> }
```

**Error:**
```json
{ "error": { "code": "<ERROR_CODE>", "message": "<human-readable>" } }
```

**HTTP status codes:** 200 (ok), 201 (created), 400 (bad request),
401 (unauthorized), 403 (forbidden), 404 (not found), 409 (conflict).

## Health Check

```
GET /health
```

No authentication required. Returns `200` with:
```json
{ "status": "ok" }
```

Used by Janissary and Vizier to wait for Divan readiness at startup.

---

## Provinces

### Create Province

```
POST /provinces
```

```json
{
  "id": "prov-a1b2c3",
  "name": "backend-refactor",
  "ip": "10.13.13.5",
  "status": "creating",
  "firman": "openclaw-firman",
  "berat": "openclaw-coding-berat",
  "repo": "stranma/EFM",
  "branch": "main"
}
```

`id` is caller-provided (Vizier generates it). Returns `201` with the
created province object, or `409` if the ID already exists.

### List Provinces

```
GET /provinces
GET /provinces?status=running
```

Returns `200` with array of province objects. Optional `status` filter.
Destroyed provinces are included unless filtered.

### Get Province

```
GET /provinces/{id}
```

Returns `200` with province object, or `404`.

### Update Province

```
PATCH /provinces/{id}
```

```json
{ "status": "running" }
```

Partial update. Only `status`, `ip`, and `name` are mutable. Returns `200`
with updated province object, or `404`.

### Province Object

```json
{
  "id": "prov-a1b2c3",
  "name": "backend-refactor",
  "ip": "10.13.13.5",
  "status": "creating",
  "firman": "openclaw-firman",
  "berat": "openclaw-coding-berat",
  "repo": "stranma/EFM",
  "branch": "main",
  "created_at": "2026-04-23T10:30:00Z",
  "updated_at": "2026-04-23T10:30:00Z"
}
```

Status enum: `creating`, `running`, `stopped`, `failed`, `destroying`.

---

## Grants

### Create Grant

```
POST /grants
```

```json
{
  "province_id": "prov-a1b2c3",
  "source_ip": "10.13.13.5",
  "match": {
    "domain": "api.github.com"
  },
  "inject": {
    "header": "Authorization",
    "value": "Bearer EXAMPLE_TOKEN_HERE"
  },
  "openbao_lease_id": "auth/token/create/abcd1234",
  "lease_expires_at": "2026-04-24T10:30:00Z"
}
```

`id` is auto-generated. `province_id` must reference an existing
province. `openbao_lease_id` references the OpenBao lease that backs
this credential; `lease_expires_at` is the server-side TTL expiry.
Both are optional at the API level but should be set for every grant
Aga creates (omitted only for tokens Aga imported from outside OpenBao,
if ever). Returns `201` with the created grant object.

### List Grants by Source

```
GET /grants?source_ip=10.13.13.5
```

Returns `200` with array of grant objects for the given source IP.
`inject.value` is masked (`"***"`) for non-authorized roles.

### Delete Grant

```
DELETE /grants/{id}
```

Returns `200` on success, `404` if not found. The Divan record is
removed; the OpenBao lease is revoked separately by Aga (Divan does
not talk to OpenBao).

### Delete All Grants for Province

```
DELETE /grants?province_id=prov-a1b2c3
```

Returns `200` with `{ "data": { "deleted": <count> } }`.

### Grant Object

```json
{
  "id": "grant-x1y2z3",
  "province_id": "prov-a1b2c3",
  "source_ip": "10.13.13.5",
  "match": {
    "domain": "api.github.com"
  },
  "inject": {
    "header": "Authorization",
    "value": "Bearer EXAMPLE_TOKEN_HERE"
  },
  "openbao_lease_id": "auth/token/create/abcd1234",
  "lease_expires_at": "2026-04-24T10:30:00Z",
  "created_at": "2026-04-23T10:31:00Z"
}
```

### Lease Expiry Semantics

Janissary MUST compare `lease_expires_at` against current UTC time
before injecting. If the lease has expired:

- Skip injection (fail closed).
- Log an audit entry with `credential_injected: false` and
  `reason: "lease_expired"`.
- The request itself is not blocked (the upstream service will
  respond with a `401`/`403`, signalling to the agent that the token
  is stale); Aga sees the audit entry and re-issues.

Aga renews leases before expiry by re-reading the OpenBao lease,
updating `inject.value` and `lease_expires_at` in place via
`PATCH /grants/{id}`.

### Update Grant (Lease Renewal)

```
PATCH /grants/{id}
```

```json
{
  "inject": { "value": "Bearer NEW_TOKEN" },
  "lease_expires_at": "2026-04-25T10:30:00Z"
}
```

Aga role only. Partial update. Returns `200` with the updated grant
object.

---

## Whitelists

### Get Whitelist for Source

```
GET /whitelists/{source_id}
```

`source_id` is a province ID or `"vizier"`. Returns `200` with:

```json
{
  "data": {
    "source_id": "prov-a1b2c3",
    "domains": ["github.com", "api.github.com", "pypi.org"]
  }
}
```

Returns `200` with empty `domains` array if no whitelist exists (not 404).

### Set Whitelist for Source

```
PUT /whitelists/{source_id}
```

```json
{
  "domains": ["github.com", "api.github.com", "pypi.org"]
}
```

Full replacement. Returns `200` with the updated whitelist.

### Add Domain to Whitelist

```
POST /whitelists/{source_id}/domains
```

```json
{ "domain": "registry.npmjs.org" }
```

Appends if not present. Returns `200`. Idempotent.

### Remove Domain from Whitelist

```
DELETE /whitelists/{source_id}/domains/{domain}
```

Returns `200`. Idempotent (returns 200 even if domain wasn't present).

---

## Blacklist

### Get Blacklist

```
GET /blacklist
```

Returns `200` with:
```json
{
  "data": {
    "domains": ["paste.mozilla.org", "pastebin.com"]
  }
}
```

### Add Domain to Blacklist

```
POST /blacklist/domains
```

```json
{ "domain": "evil-paste.com" }
```

Idempotent. Returns `200`.

### Remove Domain from Blacklist

```
DELETE /blacklist/domains/{domain}
```

Idempotent. Returns `200`.

### Blacklist vs Whitelist Priority

Blacklist is checked BEFORE whitelist. A blacklisted domain is blocked
even if it appears on a source's whitelist. This prevents a compromised
subdomain from being accessible just because the parent is whitelisted.

**Traffic rule order:**
1. Blacklist -- block (any method)
2. Whitelist -- pass (any method)
3. Non-whitelist read (GET/HEAD) -- pass
4. Non-whitelist write (POST/PUT/PATCH/DELETE) -- block; eligible
   for appeal (Kashif-triaged)

---

## Appeals

### Create Appeal

```
POST /appeals
```

```json
{
  "source_ip": "10.13.13.5",
  "province_id": "prov-a1b2c3",
  "url": "https://example.com/api/data",
  "method": "POST",
  "justification": "Need to submit the PR review comment"
}
```

`id` auto-generated. `status` defaults to `pending`. `kashif_verdict`
defaults to `null`. Returns `201`.

### List Appeals

```
GET /appeals?status=pending
GET /appeals?province_id=prov-a1b2c3
GET /appeals?kashif_verdict=escalate
```

Returns `200` with array of appeal objects. Filterable by `status`,
`province_id`, and `kashif_verdict`.

### Set Kashif Verdict

```
PATCH /appeals/{id}/kashif_verdict
```

```json
{
  "kashif_verdict": "allow",
  "screened_at": "2026-04-23T11:00:05Z",
  "notes": "Regex pass clean; Prompt Guard 2 score 0.03; Llama Guard 3 benign"
}
```

Kashif role only.

`kashif_verdict` enum: `allow`, `block`, `escalate`.

- `allow`: Divan automatically transitions the appeal to
  `status: approved, decision: one-time` (Janissary picks it up
  on next poll).
- `block`: Divan automatically transitions to
  `status: denied`.
- `escalate`: appeal stays at `status: pending` for Vizier to relay
  to Sultan.

If Kashif never writes a verdict (its LLM is down or times out),
Janissary treats the appeal as `escalate` after a timeout configured
in Janissary's `config.yaml`.

### Resolve Appeal (Sultan)

```
PATCH /appeals/{id}
```

```json
{
  "status": "approved",
  "decision": "one-time"
}
```

`status`: `approved` or `denied`.
`decision` (when approved): `one-time` or `whitelist`.

When `decision` is `whitelist`, the domain from the appeal URL is
automatically added to the source's whitelist in Divan.

When `decision` is `one-time`, the appeal is marked approved. The
agent retries the request. Janissary checks Divan for approved
appeals matching the URL + method + source_ip within a configurable
(default: 5 minutes) window and lets it through. The one-time
approval window defaults to 5 minutes and can be configured via
Janissary's `config.yaml` (`appeal.one_time_timeout_minutes`).
Divan filters approved appeals by this window when serving the
`/janissary/state` endpoint.

Returns `200` with updated appeal object.

### Appeal Object

```json
{
  "id": "appeal-m1n2o3",
  "source_ip": "10.13.13.5",
  "province_id": "prov-a1b2c3",
  "url": "https://example.com/api/data",
  "method": "POST",
  "justification": "Need to submit the PR review comment",
  "status": "pending",
  "decision": null,
  "kashif_verdict": null,
  "kashif_notes": null,
  "screened_at": null,
  "created_at": "2026-04-23T11:00:00Z",
  "resolved_at": null
}
```

Status enum: `pending`, `approved`, `denied`.
Decision enum: `null`, `one-time`, `whitelist`.
Kashif verdict enum: `null`, `allow`, `block`, `escalate`.

---

## Port Requests

Non-HTTP port declarations from berats, requiring Sultan approval.

### Create Port Request

```
POST /port_requests
```

```json
{
  "province_id": "prov-a1b2c3",
  "host": "github.com",
  "port": 22,
  "protocol": "tcp",
  "reason": "Git SSH"
}
```

`status` defaults to `pending`. Returns `201`.

### List Port Requests

```
GET /port_requests?status=pending
GET /port_requests?province_id=prov-a1b2c3
```

### Resolve Port Request

```
PATCH /port_requests/{id}
```

```json
{ "status": "approved" }
```

Status: `approved` or `denied`. Aga reads approved requests and
opens the network route.

### Port Request Object

```json
{
  "id": "port-p1q2r3",
  "province_id": "prov-a1b2c3",
  "host": "github.com",
  "port": 22,
  "protocol": "tcp",
  "reason": "Git SSH",
  "status": "pending",
  "created_at": "2026-04-23T11:05:00Z",
  "resolved_at": null
}
```

---

## Audit

Janissary, Aga, and Kashif write audit entries. The dashboard reads them.

### Append Audit Entry

```
POST /audit
```

```json
{
  "component": "janissary",
  "province_id": "prov-a1b2c3",
  "source_ip": "10.13.13.5",
  "action": "http_request",
  "verdict": "allow",
  "rule": "whitelist",
  "detail": {
    "method": "POST",
    "url": "https://api.github.com/repos/stranma/EFM/pulls",
    "credential_injected": true,
    "grant_id": "grant-x1y2z3"
  }
}
```

Any writer role (`janissary`, `aga`, `kashif`, `vizier`) can append.
`id` and `created_at` are auto-generated. Returns `201`.

### List Audit Entries

```
GET /audit
GET /audit?component=kashif
GET /audit?province_id=prov-a1b2c3
GET /audit?since=2026-04-23T00:00:00Z&limit=200
```

`limit` defaults to 100, max 1000. Dashboard role only.

### Audit Entry Object

```json
{
  "id": "audit-u1v2w3",
  "component": "janissary",
  "province_id": "prov-a1b2c3",
  "source_ip": "10.13.13.5",
  "action": "http_request",
  "verdict": "allow",
  "rule": "whitelist",
  "detail": { /* free-form object */ },
  "created_at": "2026-04-23T11:00:00Z"
}
```

`component` enum: `janissary`, `kashif`, `aga`, `vizier`.
`verdict` enum: `allow`, `block`, `escalate`, `error`.

---

## Bulk State Endpoint (Janissary)

Janissary needs all its state in one call to avoid 5+ requests per poll.

```
GET /janissary/state
```

Returns `200` with:
```json
{
  "data": {
    "provinces": [ { "id": "...", "ip": "...", "status": "..." } ],
    "grants": [
      {
        "source_ip": "...",
        "match": {...},
        "inject": {...},
        "openbao_lease_id": "...",
        "lease_expires_at": "..."
      }
    ],
    "whitelists": { "prov-a1b2c3": ["github.com", "..."], "vizier": ["..."] },
    "blacklist": ["paste.mozilla.org", "..."],
    "approved_appeals": [ { "source_ip": "...", "url": "...", "method": "...", "resolved_at": "..." } ]
  }
}
```

`approved_appeals` only includes `one-time` approvals (from either
Kashif `allow` verdicts or Sultan-approved `one-time` decisions) from
the last configurable window (default: 5 minutes, set via
Janissary's `appeal.one_time_timeout_minutes`). Janissary-role only.

---

## Dashboard Routes

The dashboard is server-rendered HTML (Jinja2 + HTMX) hosted on the
same FastAPI process as the JSON API, but bound to `127.0.0.1:8601`
only. It uses HTTP basic auth (a single operator user, password set
at deploy time and stored in `/opt/sultanate/dashboard.env`). Internally
the dashboard calls the JSON API using the `dashboard` role key (grant
values always masked).

| Path | Content |
|------|---------|
| `/` | Realm: province list with status, last activity, alert count |
| `/province/{id}` | Full province detail (status, firman+berat, grants, whitelist, audit, appeals) |
| `/province/{id}/secrets` | Grants only: domain, header, `openbao_lease_id`, `lease_expires_at`, age |
| `/province/{id}/whitelist` | Effective whitelist (berat defaults + Sultan additions) |
| `/appeals` | All appeals with Kashif verdict and Sultan decision |
| `/audit` | Recent audit entries; filterable |
| `/blacklist` | Global blacklist |
| `/health` | Health status of all components |

All pages are read-only in MVP. Mutations (whitelist add, appeal
approval, grant revocation) go through Aga / Sultan via Telegram,
not through the dashboard.

---

## SQLite Schema

```sql
CREATE TABLE provinces (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    ip TEXT,
    status TEXT NOT NULL DEFAULT 'creating',
    firman TEXT NOT NULL,
    berat TEXT NOT NULL,
    repo TEXT NOT NULL,
    branch TEXT NOT NULL DEFAULT 'main',
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);

CREATE TABLE grants (
    id TEXT PRIMARY KEY,
    province_id TEXT NOT NULL REFERENCES provinces(id),
    source_ip TEXT NOT NULL,
    match_domain TEXT NOT NULL,
    inject_header TEXT NOT NULL,
    inject_value TEXT NOT NULL,
    openbao_lease_id TEXT,
    lease_expires_at TEXT,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
CREATE INDEX idx_grants_source_ip ON grants(source_ip);
CREATE INDEX idx_grants_province_id ON grants(province_id);
CREATE INDEX idx_grants_lease_expires_at ON grants(lease_expires_at);

CREATE TABLE whitelists (
    source_id TEXT NOT NULL,
    domain TEXT NOT NULL,
    PRIMARY KEY (source_id, domain)
);

CREATE TABLE blacklist (
    domain TEXT PRIMARY KEY
);

CREATE TABLE appeals (
    id TEXT PRIMARY KEY,
    source_ip TEXT NOT NULL,
    province_id TEXT NOT NULL REFERENCES provinces(id),
    url TEXT NOT NULL,
    method TEXT NOT NULL,
    justification TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    decision TEXT,
    kashif_verdict TEXT,
    kashif_notes TEXT,
    screened_at TEXT,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    resolved_at TEXT
);
CREATE INDEX idx_appeals_status ON appeals(status);
CREATE INDEX idx_appeals_province_id ON appeals(province_id);
CREATE INDEX idx_appeals_kashif_verdict ON appeals(kashif_verdict);

CREATE TABLE port_requests (
    id TEXT PRIMARY KEY,
    province_id TEXT NOT NULL REFERENCES provinces(id),
    host TEXT NOT NULL,
    port INTEGER NOT NULL,
    protocol TEXT NOT NULL DEFAULT 'tcp',
    reason TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    resolved_at TEXT
);
CREATE INDEX idx_port_requests_status ON port_requests(status);

CREATE TABLE audit (
    id TEXT PRIMARY KEY,
    component TEXT NOT NULL,
    province_id TEXT,
    source_ip TEXT,
    action TEXT NOT NULL,
    verdict TEXT NOT NULL,
    rule TEXT,
    detail_json TEXT,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
CREATE INDEX idx_audit_created_at ON audit(created_at DESC);
CREATE INDEX idx_audit_component ON audit(component);
CREATE INDEX idx_audit_province_id ON audit(province_id);

CREATE TABLE api_keys (
    key_hash TEXT PRIMARY KEY,
    role TEXT NOT NULL,
    component TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
```

API keys are stored as SHA-256 hashes. Plaintext keys only exist in
`/opt/sultanate/divan.env` (root-readable only).

## Configuration

Divan reads from environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `DIVAN_API_HOST` | `0.0.0.0` | JSON API listen address (internal Docker network) |
| `DIVAN_API_PORT` | `8600` | JSON API listen port |
| `DIVAN_DASHBOARD_HOST` | `127.0.0.1` | Dashboard listen address (host-localhost only) |
| `DIVAN_DASHBOARD_PORT` | `8601` | Dashboard listen port |
| `DIVAN_DB` | `/opt/sultanate/divan.db` | SQLite database path |
| `DIVAN_ENV_FILE` | `/opt/sultanate/divan.env` | Component API keys file |
| `DIVAN_DASHBOARD_ENV_FILE` | `/opt/sultanate/dashboard.env` | Dashboard basic-auth credentials file |

## Concurrency

SQLite in WAL mode. FastAPI with a single writer lock for mutations.
Concurrent reads are safe. Write conflicts return `409`.
