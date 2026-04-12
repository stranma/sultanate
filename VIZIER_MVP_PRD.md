# PRD: Vizier MVP -- Province Orchestration for Sultanate

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).

## What Vizier Is

CLI tool + Hermes agent that manages provinces (isolated containers). Creates
provinces from firmans (container templates) and berats (agent profiles),
tracks the realm, writes province state to Divan. Talks to Sultan via
Telegram. Routes through Janissary for all outbound traffic.

Vizier is reactive -- it acts on Sultan's commands. It does not invent work.

## Two Interfaces, One System

**CLI** (`vizier`) -- the real implementation. Manages Docker containers,
writes to Divan, applies firmans and berats.

**Hermes agent** -- conversational wrapper. Sultan talks to Vizier via
Telegram. Vizier uses the `terminal` tool to run `vizier` CLI commands.
Also polls Divan for pending appeals and relays them to Sultan.

The CLI is the first deliverable. The Hermes agent is a thin layer on top.

## Privileges

| Property | Value |
|----------|-------|
| **User** | `vizier` (dedicated, non-root) |
| **Groups** | Docker group |
| **Network** | Through Janissary (HTTP_PROXY) |
| **Can** | Create/manage containers, `docker exec` into provinces, write to Divan (provinces, berat port requests), read Divan (appeals) |
| **Cannot** | Read grant values, access Infisical, modify network rules, open ports, read secrets |

Vizier has the Janissary security MCP tool. If Vizier hits a blocked request,
it can appeal or ask Sultan to tell Sentinel to add a permanent exception.

## CLI

```
vizier create <firman> --berat <berat> --repo <repo> [--name <name>]
vizier list
vizier status <province>
vizier stop <province>
vizier start <province>
vizier destroy <province>
vizier logs <province>
```

All commands write state changes to Divan.

## Province Lifecycle

States: `creating` -> `running` -> `stopped` -> `destroying`

Failed startup: `creating` -> `failed`

Vizier writes every state change to Divan. Sentinel watches Divan for new
provinces and provisions grants accordingly.

## Province Creation Flow

```
1. Sultan: "vizier create hermes-firman --berat hermes-coding-berat --repo stranma/EFM"
2. Vizier creates container from hermes-firman Docker image
   -> internal Docker network only (no external route)
   -> HTTP_PROXY / HTTPS_PROXY pointing to Janissary
3. Vizier writes to Divan: province ID, IP, status=creating, firman, berat
4. Vizier writes berat's non-HTTP port declarations to Divan as requests
5. Sentinel sees new province in Divan:
   -> provisions default HTTP grants (tokens via Infisical, grants to Divan)
   -> asks Sultan to approve any non-HTTP port requests
   -> on approval, opens host:port pairs and provisions service tokens
6. Vizier runs workspace bootstrap inside container (repo clone via Janissary)
7. Vizier applies berat: writes SOUL.md, AGENTS.md, config.yaml
8. Vizier starts Hermes gateway inside the container
9. Vizier updates Divan: status=running
10. Sultan can now talk to the Pasha via Telegram
```

## Appeal Relay

Vizier polls Divan for pending appeals and relays them to Sultan:

1. Vizier polls `GET /appeals?status=pending`
2. For each pending appeal, sends Sultan a Telegram message:
   "Province X wants to POST to example.com -- reason: [justification].
   Approve / deny / whitelist?"
3. Sultan replies
4. Vizier writes decision to Divan (`PUT /appeals/{id}`)
5. For whitelist or grant changes, Sultan tells Sentinel

## Hermes Configuration (Vizier Agent)

Vizier runs on the upstream `nousresearch/hermes-agent` Docker image -- no
custom Dockerfile. Differentiated by configuration only:

- **Soul**: realm manager, province lifecycle, appeal relay
- **Tools**: terminal (for `vizier` CLI), Janissary security MCP
- **Telegram**: dedicated bot, Sultan-only access
- **Egress**: through Janissary

Vizier's berat ships in the vizier repo.

## Phase 1 Scope

**In scope:**
- CLI: create, list, status, stop, start, destroy, logs
- Province lifecycle with Divan state tracking
- Firman + berat based province creation
- Hermes agent wrapper with Telegram
- Appeal relay from Divan to Sultan
- Write berat port declarations to Divan as requests (Sentinel opens them)
- One firman: hermes-firman
- One berat: hermes-coding-berat

**Deferred:**
- OpenClaw support
- Cross-province coordination
- Multiple firmans
- Cost and budget reporting
- Province resource limits and quotas
