# Divan API Specification

> Technical contract for the shared state store. All components depend on
> this interface. See [SULTANATE_MVP.md](SULTANATE_MVP.md) for context.

## Overview

SQLite database + Python HTTP API (FastAPI). Listens on `127.0.0.1:8600`.
All communication is JSON over HTTP. No TLS (localhost only).

## Authentication

Pre-shared API keys, one per component. Passed as `Authorization: Bearer <key>`.
Keys are generated at deploy time and stored in `/opt/sultanate/divan.env`.

Each key maps to a role:

| Role | Key env var | Permissions |
|------|-------------|-------------|
| `vizier` | `DIVAN_KEY_VIZIER` | Read/write provinces, read/write whitelists, read appeals, write appeal decisions |
| `sentinel` | `DIVAN_KEY_SENTINEL` | Read provinces, read/write grants (including secret values), read/write whitelists, read/write blacklist |
| `janissary` | `DIVAN_KEY_JANISSARY` | Read provinces, read grants (including secret values), read whitelists, read blacklist, write appeals, read appeal decisions |

Grant secret values (`inject.value`) are only returned for `sentinel` and
`janissary` roles. Other roles see `inject.value: "***"` in responses.

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
  "ip": "172.18.0.5",
  "status": "creating",
  "firman": "hermes-firman",
  "berat": "hermes-coding-berat",
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
  "ip": "172.18.0.5",
  "status": "creating",
  "firman": "hermes-firman",
  "berat": "hermes-coding-berat",
  "repo": "stranma/EFM",
  "branch": "main",
  "created_at": "2026-04-12T10:30:00Z",
  "updated_at": "2026-04-12T10:30:00Z"
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
  "source_ip": "172.18.0.5",
  "match": {
    "domain": "api.github.com"
  },
  "inject": {
    "header": "Authorization",
    "value": "Bearer ghp_xxxxxxxxxxxx"
  }
}
```

`id` is auto-generated. `province_id` must reference an existing province.
Returns `201` with the created grant object.

### List Grants by Source

```
GET /grants?source_ip=172.18.0.5
```

Returns `200` with array of grant objects for the given source IP.
`inject.value` is masked (`"***"`) for non-authorized roles.

### Delete Grant

```
DELETE /grants/{id}
```

Returns `200` on success, `404` if not found.

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
  "source_ip": "172.18.0.5",
  "match": {
    "domain": "api.github.com"
  },
  "inject": {
    "header": "Authorization",
    "value": "Bearer ghp_xxxxxxxxxxxx"
  },
  "created_at": "2026-04-12T10:31:00Z"
}
```

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

Blacklist is checked BEFORE whitelist. A blacklisted domain is blocked even
if it appears on a source's whitelist. This prevents a compromised subdomain
from being accessible just because the parent is whitelisted.

**Updated traffic rule order:**
1. Blacklist -- block (any method)
2. Whitelist -- pass (any method)
3. Non-whitelist read (GET/HEAD) -- pass
4. Non-whitelist write (POST/PUT/PATCH/DELETE) -- block

---

## Appeals

### Create Appeal

```
POST /appeals
```

```json
{
  "source_ip": "172.18.0.5",
  "province_id": "prov-a1b2c3",
  "url": "https://example.com/api/data",
  "method": "POST",
  "justification": "Need to submit the PR review comment"
}
```

`id` auto-generated. `status` defaults to `pending`. Returns `201`.

### List Appeals

```
GET /appeals?status=pending
GET /appeals?province_id=prov-a1b2c3
```

Returns `200` with array of appeal objects. Filterable by `status` and
`province_id`.

### Resolve Appeal

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

When `decision` is `one-time`, the appeal is marked approved. The agent
retries the request. Janissary checks Divan for approved appeals matching
the URL + method + source_ip within a 5-minute window and lets it through.

Returns `200` with updated appeal object.

### Appeal Object

```json
{
  "id": "appeal-m1n2o3",
  "source_ip": "172.18.0.5",
  "province_id": "prov-a1b2c3",
  "url": "https://example.com/api/data",
  "method": "POST",
  "justification": "Need to submit the PR review comment",
  "status": "pending",
  "decision": null,
  "created_at": "2026-04-12T11:00:00Z",
  "resolved_at": null
}
```

Status enum: `pending`, `approved`, `denied`.
Decision enum: `null`, `one-time`, `whitelist`.

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

Status: `approved` or `denied`. Sentinel reads approved requests and
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
  "created_at": "2026-04-12T11:05:00Z",
  "resolved_at": null
}
```

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
    "grants": [ { "source_ip": "...", "match": {...}, "inject": {...} } ],
    "whitelists": { "prov-a1b2c3": ["github.com", "..."], "vizier": ["..."] },
    "blacklist": ["paste.mozilla.org", "..."],
    "approved_appeals": [ { "source_ip": "...", "url": "...", "method": "...", "resolved_at": "..." } ]
  }
}
```

`approved_appeals` only includes `one-time` approvals from the last
5 minutes (the retry window). Janissary-role only.

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
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
);
CREATE INDEX idx_grants_source_ip ON grants(source_ip);
CREATE INDEX idx_grants_province_id ON grants(province_id);

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
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
    resolved_at TEXT
);
CREATE INDEX idx_appeals_status ON appeals(status);
CREATE INDEX idx_appeals_province_id ON appeals(province_id);

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
| `DIVAN_HOST` | `127.0.0.1` | Listen address |
| `DIVAN_PORT` | `8600` | Listen port |
| `DIVAN_DB` | `/opt/sultanate/divan.db` | SQLite database path |
| `DIVAN_ENV_FILE` | `/opt/sultanate/divan.env` | API keys file |

## Concurrency

SQLite in WAL mode. FastAPI with a single writer lock for mutations.
Concurrent reads are safe. Write conflicts return `409`.
