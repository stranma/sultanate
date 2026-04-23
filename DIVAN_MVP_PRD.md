# PRD: Divan MVP -- Shared State Store and Dashboard

> For shared glossary, deployment model, and component overview see
> [SULTANATE_MVP.md](SULTANATE_MVP.md). For HTTP contract detail see
> [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md).

## What Divan Is

Divan is Sultanate's shared state store. A single SQLite file plus a
FastAPI HTTP API, with a server-rendered read-only web dashboard mounted
on the same process.

Divan is **not** an orchestrator. It stores records and serves them.
Vizier, Janissary, Kashif, Aga all read from and write to Divan; no
component calls another directly. Coordination happens through Divan.

Divan ships with Janissary in the same repo (`janissary`) but runs as a
separate container in `docker-compose.yml`.

## What Divan Holds

- **Province registry** -- ID, IP, name, status, firman, berat, repo
- **Grant table** -- source IP + destination -> header injection rule,
  with OpenBao `lease_id` and `lease_expires_at`
- **Whitelists** -- per-source domain allowlists
- **Blacklist** -- global blocked domains (curated by Aga)
- **Appeals** -- pending/resolved appeal records with Kashif verdicts
- **Port requests** -- non-HTTP port access requests
- **Audit log** -- decisions from Janissary, Aga, Kashif

Schemas and endpoints are in `DIVAN_API_SPEC.md`.

## What Divan Does NOT Do

- No orchestration -- Divan never initiates actions
- No authentication of Sultan -- only component-to-Divan API keys
- No business logic -- rule evaluation lives in Janissary and Kashif
- No push notifications -- consumers poll
- No secret storage -- dangerous secrets live in OpenBao; Divan stores
  lease IDs and injection rules only

## Access Control

Pre-shared API keys, one per component, generated at deploy time. Each
key maps to a role with endpoint-level read/write permissions. Grant
secret values (the `inject.value` field) are only returned to the
Janissary role. The dashboard role reads everything but writes nothing.

See `DIVAN_API_SPEC.md` for the role matrix.

## Dashboard

Server-rendered pages using Jinja2 + HTMX, hosted in the same FastAPI
process as the API. No SPA framework, no separate frontend service.

**Pages (MVP):**

| Path | Content | Writes |
|------|---------|--------|
| `/` (Realm) | Province list: status, firman, berat, last activity, active-grant count, pending-appeal count | none |
| `/province/{id}` | Full province detail: status, IP, firman+berat, Pasha identity, whitelist, active grants, recent audit, pending appeals | none |
| `/province/{id}/secrets` | Grants only: domain, header, `openbao_lease_id`, `lease_expires_at`, age | none (revocation requires calling Aga) |
| `/province/{id}/whitelist` | Effective whitelist: berat defaults + Sultan additions | none (whitelist edits require Aga instruction) |
| `/appeals` | All appeals: pending, auto-decided by Kashif, escalated to Aga/Sultan | none |
| `/audit` | Recent audit entries, filterable by province / action / verdict | none |
| `/blacklist` | Global blacklist curated by Aga | none |
| `/health` | Health status of all components (OpenBao, Divan, Janissary, Kashif, Aga, Vizier) | none |

The dashboard is **read-only** in MVP. Any mutation (whitelist add,
appeal approval, grant revocation) goes through Aga / Sultan /
Telegram. The dashboard exists so Sultan can see state at a glance,
not to replace the command path.

**Auth + network:**

- HTTP basic auth (single user: Sultan, password from OpenBao at boot)
- Listener bound to `127.0.0.1:8601`
- Sultan reaches the dashboard via SSH tunnel
  (`ssh -L 8601:127.0.0.1:8601 sultan@host`, then
  `http://localhost:8601` in browser)

## Implementation Stack

- Python 3.12+
- FastAPI + Jinja2 + HTMX
- SQLite (WAL mode, single writer per connection; FastAPI serializes
  via per-request sessions)
- No ORM for MVP; raw SQL via `sqlite3` stdlib is sufficient
- Container: `python:3.12-slim` base, requirements frozen in
  `requirements.txt`

## Startup

Divan starts **before** Janissary, Kashif, Aga, Vizier. After OpenBao.
On boot:

1. Ensure SQLite file exists; run migrations (create tables if new).
2. Bind FastAPI to `0.0.0.0:8600` (API) -- reachable from other
   Sultanate containers on the internal Docker network.
3. Bind dashboard to `127.0.0.1:8601` (host-localhost only; Sultan
   tunnels in).
4. `/health` returns 200 when both listeners are up and SQLite is
   readable/writable.

## Fail-Closed Behavior

If Divan is unreachable:

- **Janissary** serves last-cached rules. If no cache (first boot),
  blocks all traffic.
- **Vizier** returns an error on any `vizier-cli` command that needs
  state.
- **Aga** alerts Sultan via Telegram.
- **Kashif** continues to screen (it's stateless) but its verdicts
  are buffered in-process until Divan returns.

## Phase 1 Scope

**In scope:**
- All API endpoints listed in `DIVAN_API_SPEC.md`
- Role-based access control via pre-shared keys
- Dashboard pages listed above (read-only)
- HTTP basic auth + 127.0.0.1 binding
- `/health` endpoint
- SQLite single-file storage with daily on-host backups

**Deferred:**
- Mutations from the dashboard (all mutations go through Aga)
- Charts and live metrics (page refresh only)
- Dark mode, theming, responsive layout niceties
- Multi-operator access
- PostgreSQL migration
- Signed audit records (per-entry ECDSA or HMAC chaining)
- Dual audit sinks (streaming export to a second collector)
- Off-host backup automation
- Authentication beyond HTTP basic auth (OAuth/OIDC/TLS client certs)
