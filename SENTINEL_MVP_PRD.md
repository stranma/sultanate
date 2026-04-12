# PRD: Sentinel MVP -- Secret Management for Sultanate

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).

## What Sentinel Is

A trusted Hermes agent running as root on the host. Manages dangerous secrets
(GitHub tokens, API keys) and provisions grants so Janissary can inject
credentials at the proxy level. Talks to Sultan via Telegram.

Sentinel is trusted and instructed, not constrained. It has full host access
but follows policy: dangerous secrets go through Janissary's grant table,
never into containers.

## Responsibilities

1. **Secret provisioning** -- create tokens in Infisical, write grants to
   Divan so Janissary can inject them into request headers
2. **Grant lifecycle** -- provision default grants when a new province
   appears in Divan, revoke all grants when a province is destroyed
3. **Non-HTTP access** -- read berat port declarations from Divan, ask
   Sultan for approval via Telegram, then open specific host:port pairs
   via iptables/Docker network rules (requires root) and provision service
   tokens. No auto-approve -- every port opening requires Sultan's decision.
4. **Whitelist management** -- update per-source whitelists in Divan on
   Sultan's instruction
5. **Blacklist management** -- update the global blacklist in Divan on
   Sultan's instruction

## What Sentinel Does NOT Do (MVP)

- No alert contextualization (Sultan reads raw appeals via Vizier)
- No automated access request review (Sultan decides directly)
- No audit queries (deferred)
- No blacklist curation from traffic patterns (manual only)
- No content inspection (deferred: Kashif)

## Trust Model

| Property | Value |
|----------|-------|
| **User** | root |
| **Network** | Direct host networking (no Janissary) |
| **Egress** | Telegram API + Infisical (local) only |
| **Ingress** | Telegram (Sultan) only. No province traffic. |
| **Divan access** | Read provinces, read/write grants, read/write whitelists, read/write blacklist |

Sentinel has no exposure to province content. No agent output, no appeal
payloads, no fetched web content reaches Sentinel. This eliminates the need
for Kashif input screening in MVP.

## Grant Lifecycle

```
1. Vizier creates province, writes to Divan (ID, IP, berat, status=creating)
2. Sentinel polls Divan, sees new province
3. Sentinel reads berat's default grants (e.g., GitHub read/write for repo X)
4. Sentinel provisions tokens in Infisical
5. Sentinel writes grant rules to Divan:
   { source_ip, domain, header, value }
6. Sentinel writes berat default whitelist to Divan
7. Janissary reads grants + whitelist from Divan, starts injecting

Province destroyed:
8. Vizier updates Divan: status=destroying
9. Sentinel sees status change, revokes tokens in Infisical
10. Sentinel deletes grants from Divan
11. Janissary stops injecting on next poll
```

## Sultan Interactions

Sultan talks to Sentinel via a dedicated Telegram bot:

- "Add github.com/stranma/new-repo write access for province X"
  -> Sentinel provisions token, writes grant to Divan
- "Revoke province X's GitHub access"
  -> Sentinel revokes token in Infisical, deletes grant from Divan
- "Whitelist example.com for province X"
  -> Sentinel writes to Divan whitelists
- "Add badsite.com to the blacklist"
  -> Sentinel writes to Divan blacklist

## Infisical Integration

Sentinel uses Infisical as the secret vault:

- Stores actual secret values (tokens, API keys)
- Scoped per-province (environment or path-based)
- Sentinel reads values from Infisical when writing grants to Divan
- On revocation, Sentinel deletes from both Infisical and Divan

Infisical runs locally on the host. Sentinel accesses it directly (no proxy).

## Hermes Configuration

Sentinel runs on the upstream `nousresearch/hermes-agent` Docker image --
no custom Dockerfile. Differentiated by configuration only:

- **Soul**: trusted operator agent, manages secrets, follows policy
- **Tools**: terminal (for infisical CLI), Divan HTTP API access
- **Telegram**: dedicated bot, Sultan-only access
- **Model**: same as province agents (configurable)

Sentinel's berat is built-in (ships with the janissary repo), not a
separate repo.

## Phase 1 Scope

**In scope:**
- Secret provisioning via Infisical
- Grant lifecycle tied to province lifecycle via Divan
- Whitelist and blacklist management via Sultan commands
- Telegram communication with Sultan
- Polling Divan for new/destroyed provinces

**Deferred:**
- Alert contextualization (Vizier relays raw appeals for now)
- Automated access request review
- Audit queries
- Traffic pattern analysis and blacklist curation
- Kashif input screening
- Grant TTL and automatic expiry
