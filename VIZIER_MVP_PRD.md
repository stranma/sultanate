# PRD: Vizier MVP -- Province Orchestration for Sultanate

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For detailed implementation see [VIZIER_SPEC.md](VIZIER_SPEC.md).

## What Vizier Is

CLI tool + OpenClaw agent that manages provinces (isolated containers).
Creates provinces from firmans (container templates) and berats (agent
profiles), tracks the realm, writes province state to Divan. Talks to
Sultan via Telegram. Routes through Janissary for all outbound traffic.

Vizier is reactive -- it acts on Sultan's commands. It does not invent
work.

## Two Interfaces, One System

**CLI** (`vizier-cli`) -- the real implementation. Manages Docker
containers, writes to Divan, applies firmans and berats.

**OpenClaw agent** -- conversational wrapper. Sultan talks to Vizier via
Telegram. Vizier uses OpenClaw's `bash` tool to run `vizier-cli` commands.
Also polls Divan for pending appeals (escalated by Kashif) and relays
them to Sultan.

The CLI is the first deliverable. The OpenClaw agent is a thin layer on
top.

## Privileges

| Property | Value |
|----------|-------|
| **User** | `vizier` (dedicated, non-root) |
| **Groups** | Docker group |
| **Network -- egress** | Through Janissary (WireGuard transparent proxy), same rules as a Pasha |
| **Network -- ingress** | None. All Sultan ↔ Vizier comms via Telegram polling (outbound only). The OpenClaw gateway HTTP endpoint on port 18789 is bound to `127.0.0.1` inside the Vizier container -- never reachable from outside. |
| **Can** | Create/manage containers, `docker exec` into provinces, write to Divan (provinces, berat port requests, whitelist defaults, appeal decisions), read Divan (appeals, audit) |
| **Cannot** | Read grant values, access OpenBao, modify network rules, open ports, read secrets |

Vizier has access to the Janissary security HTTP API (appeal /
request_access). If Vizier itself hits a blocked request (egress from the
Vizier container), it can appeal the same way a Pasha would -- the
appeal is screened by Kashif first.

## Vizier Egress

Vizier needs egress for two purposes:

1. **Telegram polling** -- outbound calls to `api.telegram.org` to
   long-poll for Sultan's messages. Always allowed (whitelisted in
   Vizier's Divan whitelist).
2. **Documentation lookups** -- when Sultan asks Vizier to set up a
   province for a new repo or framework, Vizier may need to read docs
   sites, package registries, or Q&A sites to plan a sensible
   firman/berat parameterisation.

Vizier's whitelist key in Divan is `"vizier"` (parallel to province
IDs as keys). It ships with these defaults:

| Domain | Reason |
|--------|--------|
| `api.telegram.org` | Telegram polling |
| `docs.python.org` | Python documentation |
| `docs.openclaw.ai` | OpenClaw runtime documentation |
| `github.com`, `api.github.com` | Repo discovery, README lookup |
| `pypi.org`, `files.pythonhosted.org` | Python package metadata |
| `registry.npmjs.org` | Node package metadata |
| `cdn.jsdelivr.net` | Common CDN for npm |
| `docs.docker.com` | Docker reference |
| `stackoverflow.com` | Developer Q&A |

These cover Vizier's day-to-day reads. Janissary's read-only
passthrough rule (Rule 3 -- non-whitelist GET/HEAD = pass) covers
unlisted documentation sites without an explicit whitelist entry, so
Vizier can browse generally even when a domain isn't listed.

Sultan can extend Vizier's whitelist via Aga (e.g., add a new search
engine or vendor docs site) with the same Telegram instruction flow
used for province whitelist edits.

Vizier never needs **write** access (POST/PUT/PATCH/DELETE) to
non-whitelisted domains. If a write is genuinely needed, Vizier appeals
the same way a Pasha would -- through the Janissary security MCP.

## CLI

```
vizier-cli create <firman> --berat <berat> --repo <repo> [--name <name>]
vizier-cli list
vizier-cli status <province>
vizier-cli stop <province>
vizier-cli start <province>
vizier-cli destroy <province>
vizier-cli logs <province>
```

All commands write state changes to Divan.

## Province Lifecycle

States: `creating` -> `running` -> `stopped` -> `destroying`

Failed startup: `creating` -> `failed`

Vizier writes every state change to Divan. Aga watches Divan for new
provinces and provisions grants accordingly.

## Province Creation Flow

```
1. Sultan (via Telegram to Vizier):
   "create a coding agent for stranma/EFM"
2. Vizier interprets, picks firman and berat, runs:
   vizier-cli create openclaw-firman \
                     --berat openclaw-coding-berat \
                     --repo stranma/EFM
3. Vizier creates wg-client sidecar + province container pair from
   openclaw-firman image:
   -> internal Docker network only (no external route)
   -> all traffic routed through Janissary via WireGuard
4. Vizier writes to Divan: province ID, IP (WireGuard peer IP),
   status=creating, firman, berat, repo
5. Vizier writes berat's non-HTTP port declarations to Divan as
   port_requests
6. Aga sees new province in Divan:
   -> mints GitHub App installation token scoped to the repo
      (dynamic mode); writes grant to Divan with OpenBao lease ID
      and GitHub-returned expires_at
   -> or, for services without dynamic engines, asks Sultan for
      token via Telegram; stores in OpenBao KV; writes grant
      with null lease fields
   -> asks Sultan to approve any pending port_requests
   -> on approval, opens host:port pairs via iptables and provisions
      service tokens
7. Vizier runs workspace bootstrap inside container (repo clone via
   Janissary; credential injected by Janissary using the grant Aga
   just wrote)
8. Vizier applies berat inside the container:
   -> writes SOUL.md, AGENTS.md, IDENTITY.md to workspace root
   -> writes ~/.openclaw/openclaw.json
9. Vizier starts OpenClaw gateway inside the container:
   -> openclaw gateway --port 18789 in the background
10. Vizier updates Divan: status=running
11. Sultan can now talk to the Pasha via Telegram (the Pasha has its
    own OpenClaw bot, configured by the berat)
```

## Appeal Relay

Vizier polls Divan for appeals requiring Sultan's decision and relays
them. Kashif auto-decides many appeals (see
[ARCHITECTURE.md](ARCHITECTURE.md) appeal flow); Vizier only forwards
those with `kashif_verdict = escalate` (or `null` after Kashif
timeout), plus informational notices for `kashif_verdict = block`.

1. Vizier polls `GET /appeals?status=pending&kashif_verdict=escalate`
   (actionable) and
   `GET /audit?severity=alert&component=kashif&since=<last>`
   (informational -- auto-decided blocks Sultan should know about).
2. For each actionable appeal, sends Sultan a Telegram message:

   > Province X wants to POST to example.com -- reason:
   > [justification]. Kashif: escalate (notes: ...). Approve once /
   > approve forever / deny / kill province?

3. For informational Kashif blocks, sends Sultan:

   > Prov X's appeal to example.com was auto-BLOCKED by Kashif.
   > Kashif notes: ... Decision is final; pattern recurring?

4. Sultan replies to the actionable message.
5. Vizier writes decision to Divan (`PATCH /appeals/{id}`).
6. For whitelist or new-grant changes, Sultan tells Aga directly (not
   via Vizier) -- Aga has the privileges to modify those.

## OpenClaw Configuration (Vizier Agent)

Vizier runs on the upstream `openclaw/openclaw` Docker image -- no
custom Dockerfile. Differentiated by configuration only:

- **SOUL.md**: realm manager, province lifecycle, appeal relay.
- **AGENTS.md**: rules about when to call vizier-cli, how to
  format Telegram messages for Sultan, how to poll appeals.
- **Tools**: OpenClaw built-ins (`bash`, `read`, `write`, `edit`),
  Janissary security MCP server for self-appeals.
- **Telegram**: dedicated bot, Sultan-only access
  (`channels.telegram.allowFrom = <sultan-id>`).
- **Egress**: through Janissary.

Vizier's berat ships in the `vizier` repo.

## Phase 1 Scope

**In scope:**
- `vizier-cli` with create/list/status/stop/start/destroy/logs
- Province lifecycle with Divan state tracking (creating / running /
  stopped / failed / destroying)
- Firman + berat based province creation
- OpenClaw agent wrapper with Telegram
- Appeal relay from Divan to Sultan:
  - actionable for `kashif_verdict=escalate`
  - informational for `kashif_verdict=block` (audit severity=alert)
- Write berat port declarations to Divan as port_requests (Aga opens
  them on approval)
- One firman: `openclaw-firman`
- One berat: `openclaw-coding-berat`

**Deferred:**
- Additional runtimes beyond OpenClaw (OpenHands, CrewAI, custom)
  -- deferred to Phase 2 firmans
- Cross-province coordination
- Multiple firmans and berats
- Cost and budget reporting
- Province resource limits and quotas (CPU, RAM, disk per province)
- Automatic province scaling
- Multi-machine deployment (provinces across hosts)
