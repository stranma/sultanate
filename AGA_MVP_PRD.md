# PRD: Aga MVP -- Secret Management for Sultanate

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For implementation detail see [AGA_SPEC.md](AGA_SPEC.md).
> For the timeline-style flow diagrams that show where Aga sits in the
> appeal-escalation and credential-request paths, see
> [ARCHITECTURE.md](ARCHITECTURE.md) §2 (appeal flow) and §2.3
> (mid-task credential request flow).

## What Aga Is

A trusted OpenClaw agent running as root on the host. Aga is the Agha of
the Janissaries in the Ottoman metaphor -- chief of the guard corps,
commander of the security perimeter. Aga manages dangerous secrets
(GitHub tokens, API keys, database credentials) and provisions grants so
Janissary can inject credentials at the proxy level. Talks to Sultan via
Telegram.

Aga is trusted and instructed, not constrained. It has full host access
but follows policy: dangerous secrets live in OpenBao and reach Janissary
only as grant records in Divan; they never enter a province container.

## Responsibilities

1. **Secret provisioning** -- mint or retrieve tokens via OpenBao
   (GitHub App installation tokens, DB creds, SSH certs; KV fallback for
   Sultan-pasted tokens). Write grants to Divan so Janissary can inject
   them into request headers.
2. **Grant lifecycle** -- provision default grants when a new province
   appears in Divan, renew lease-bound grants before expiry while the
   province is running, revoke all grants when a province is destroyed.
3. **GitHub App integration** -- hold the Sultanate GitHub App private
   key (in OpenBao KV), mint short-lived installation tokens per-province
   scoped to the province's repo. Renew every ~15 min while the province
   is running.
4. **Non-HTTP access** -- read berat port declarations from Divan, ask
   Sultan for approval via Telegram, then open specific host:port pairs
   via iptables/Docker network rules (requires root) and provision
   service tokens. No auto-approve -- every port opening requires
   Sultan's decision.
5. **Whitelist management** -- update per-source whitelists in Divan on
   Sultan's instruction.
6. **Blacklist management** -- update the global blacklist in Divan on
   Sultan's instruction. May recommend additions to Sultan based on
   patterns observed in Divan audit (repeated Kashif blocks against
   a domain, lease-expired abuse, etc.).
7. **Appeal / access-request context** -- for escalated appeals (Kashif
   verdict = `escalate`) and Kashif-blocked events, Aga polls Divan
   audit and Telegram-messages Sultan with behavioural context
   (e.g., "prov-a1b2c3 has 3 Kashif blocks in 10 min against unfamiliar
   domains; consider destroying").

## What Aga Does NOT Do (MVP)

- No content inspection of its own (Kashif's job; all Aga's LLM-context
  ingress is pre-screened by Kashif)
- No automated access-request approval (Sultan decides; Aga adds context)
- No complex audit queries via dashboard (dashboard is read-only; Aga
  answers questions by polling Divan in the background)
- No blacklist curation from raw traffic patterns beyond simple
  counters (session-aware analysis deferred)

## Trust Model

| Property | Value |
|----------|-------|
| **User** | root |
| **Network** | Direct host networking (Aga is trusted; does not route through Janissary) |
| **Egress** | Telegram API + OpenBao (127.0.0.1) + GitHub API (for App token minting) only |
| **Ingress** | Telegram (Sultan) only. All Pasha-originated content and fetched web content is pre-screened by Kashif before reaching Aga's LLM context. |
| **Divan access** | Read provinces, read/write grants, read/write whitelists, read/write blacklist, read/write appeal decisions, read/write port_requests, write audit |
| **OpenBao access** | Aga is the sole OpenBao client. Authenticates via AppRole at startup. |

Aga's trust boundary is wider than any other component's. Sultan's one-time
GitHub App setup and Telegram commands are Aga's only inputs; everything
else flows through Divan (which is itself screened at write time by
component-role permissions).

## Grant Lifecycle (GitHub App, Phase 1 default)

```
1. Vizier creates province prov-a1b2c3 for repo stranma/EFM.
   Writes to Divan: POST /provinces
     { id, name, ip=10.13.13.5, berat=openclaw-coding-berat,
       status=creating, firman=openclaw-firman, repo=stranma/EFM }.

2. Aga polls Divan every ~5 s, sees new province.

3. Aga reads the Sultanate GitHub App private key from OpenBao KV
   (kv/github-app/private-key). The key was placed at first boot by
   the deploy script prompting Sultan.

4. Aga mints an installation access token:
   - Generate a JWT signed with the App private key
   - POST /app/installations/{installation_id}/access_tokens to GitHub
     with { repository: ["stranma/EFM"] }
   - GitHub returns { token, expires_at }  (TTL 1 hour)

5. Aga writes the grant to Divan: POST /grants
   {
     province_id: "prov-a1b2c3",
     source_ip: "10.13.13.5",
     match: { domain: "api.github.com" },
     inject: { header: "Authorization", value: "<token>" },
     openbao_lease_id: "github-app:prov-a1b2c3",
     lease_expires_at: "<GitHub's expires_at>"
   }
   Additional grant for github.com (git clone/push over HTTPS) with
   the same token.

6. Aga writes the berat's default whitelist to Divan.

7. Aga updates its internal renewal schedule: renew this grant when
   lease_expires_at < now + 20 min.

8. Janissary polls /janissary/state, picks up the new grant +
   whitelist, starts injecting on province traffic.

9. Vizier updates province status=running.

Renewal loop (every ~15 min):
   - For each active grant where lease_expires_at < now + 20 min AND
     the province status is "running":
     - Mint a fresh installation token (repeat step 4)
     - PATCH /grants/{id} with new inject.value and new
       lease_expires_at.
   - Janissary picks up the new value on its next 5 s poll.

Province destroyed:
   10. Vizier updates Divan: status=destroying
   11. Aga sees status change:
       - Stops renewing grants for this province.
       - For KV-mode grants (no lease): explicitly revokes in OpenBao
         KV, deletes grant record from Divan.
       - For dynamic-mode grants (lease-bound): revokes the underlying
         OpenBao lease and deletes the grant record. Even if Aga
         crashes here, the GitHub-App-token expires within 1 hour
         naturally -- no forgotten zombies.
   12. Janissary stops injecting on next poll.
```

## Grant Lifecycle (KV fallback)

For services without a dynamic mint path (non-GitHub, third-party APIs
Aga cannot automate against), Sultan pastes a token via Telegram once:

```
1. Sultan: "Store this for prov-a1b2c3, api.example.com:
            xyz-secret-token"
2. Aga: stores in OpenBao KV at kv/provinces/prov-a1b2c3/api.example.com
3. Aga: POST /grants with inject.value=<token>,
        openbao_lease_id=null, lease_expires_at=null
4. Janissary injects unconditionally (no expiry check for null leases)
5. Token lives until Sultan tells Aga to revoke
```

## Sultan Interactions

Sultan talks to Aga via a dedicated Telegram bot:

- **GitHub App setup (one-time):**
  - Aga at first boot: "Drop the Sultanate GitHub App private key PEM
    into /opt/sultanate/bootstrap/github-app.pem, or paste it to me
    here and I'll stash it."
  - Sultan provides the key. Aga writes it to OpenBao KV. Deletes the
    bootstrap file.

- **Day-to-day (unprompted by Sultan):**
  - Aga mints, renews, and revokes GitHub tokens automatically.

- **Sultan requests (on demand):**
  - "Add github.com/stranma/new-repo write access for prov-a1b2c3"
    -> Aga ensures the GitHub App is installed on that repo (if not,
       replies with the install URL Sultan must click);
       mints a token scoped to the repo; writes grant to Divan.
  - "Revoke prov-a1b2c3's GitHub access"
    -> Aga deletes grants from Divan; token expires naturally at
       GitHub-TTL.
  - "Whitelist example.com for prov-a1b2c3"
    -> Aga writes to Divan whitelists.
  - "Blacklist badsite.com"
    -> Aga writes to Divan blacklist.
  - "Store this token for prov-a1b2c3, <service>: <paste>"
    -> KV fallback path.

- **Unsolicited alerts from Aga:**
  - "prov-a1b2c3 has had 3 Kashif blocks in 10 min against
    unfamiliar paste sites. Consider destroying the province."
  - "prov-a1b2c3 is about to lose access to api.github.com
    (lease_expires_at in 5 min) and my renewal call failed.
    Investigating."

## OpenBao Integration

Aga is the sole OpenBao client. Authentication:

- AppRole role_id + secret_id stored in `/opt/sultanate/aga/openbao.env`
  (deploy-time generated by the deploy script; Sultan sees the values
  once and can rotate them anytime).
- At startup, Aga calls `POST /v1/auth/approle/login` with the
  role_id + secret_id and receives a client token.
- The client token is used for all subsequent OpenBao operations.

Policy (Aga's AppRole policy, configured at deploy time):

- `path "kv/github-app/*"` -- read/write/delete (for App private key)
- `path "kv/provinces/*"` -- read/write/delete (KV fallback secrets
  per province)
- `path "sys/leases/lookup"` -- read (lease introspection)
- `path "sys/leases/renew"` -- update (renew leases)
- `path "sys/leases/revoke"` -- update (revoke leases)

Explicitly denied (enforced in Aga's AppRole policy):

- `path "sys/audit/*"` -- deny (Aga must not disable/modify audit)
- `path "sys/policies/*"` -- deny (Aga must not change policies)
- `path "sys/seal"` -- deny
- `path "sys/generate-root"` -- deny
- `path "auth/approle/role/aga"` -- deny (Aga cannot rotate itself)

If OpenBao is sealed or unreachable at Aga startup, Aga retries
authentication every 10 s and posts an alert to Sultan.

## OpenClaw Configuration

Aga runs on the upstream `openclaw/openclaw` Docker image -- no custom
Dockerfile. Differentiated by configuration only:

- **`SOUL.md`**: trusted chief-of-security agent, manages secrets,
  follows policy, surfaces behavioural context to Sultan.
- **`AGENTS.md`**: instruction template with rules about OpenBao,
  Divan, Telegram, and never ingesting unscreened content.
- **Tools enabled** (OpenClaw built-ins): `bash`, `read`, `write`,
  `edit`, plus a custom MCP server (`aga-ops`) exposing
  `mint_github_token`, `revoke_grant`, `add_to_whitelist`,
  `add_to_blacklist`, `screen_and_reply` (calls Kashif first, then
  formulates Sultan reply), etc.
- **Telegram**: dedicated bot, Sultan-only access
  (`channels.telegram.allowFrom = <sultan-id>`).
- **Model**: same as Pashas (configurable), typically
  `anthropic/claude-sonnet-4`.

Aga's berat is built-in (ships with the `janissary` repo), not a
separate repo. See `AGA_SPEC.md` for full config.

## Divan Integration

Aga is tied into Divan via polling loops running inside the OpenClaw
agent:

- **Province poll** (every ~5 s): `GET /provinces?status=creating`
  -> provision grants. `GET /provinces?status=destroying` -> revoke.
- **Lease-renewal poll** (every ~15 min): iterate active grants,
  renew those expiring soon.
- **Appeal-escalation poll** (every ~10 s):
  `GET /appeals?status=pending&kashif_verdict=escalate` -> compose
  context message for Sultan and send via Telegram.
- **Kashif-block alert poll** (every ~30 s):
  `GET /audit?severity=alert&component=kashif&since=<last>` ->
  maintain per-province counter; trigger Sultan alert if threshold
  exceeded.
- **Port-request poll** (every ~10 s):
  `GET /port_requests?status=pending` -> ask Sultan, on approval
  open iptables rule, PATCH status.

## Phase 1 Scope

**In scope:**
- Secret provisioning via OpenBao (GitHub App primary + KV fallback)
- Grant lifecycle tied to province lifecycle via Divan, with lease
  renewal loop for dynamic-mode grants
- Whitelist and blacklist management via Sultan commands
- Telegram communication with Sultan (dedicated bot)
- Polling Divan for new/destroyed provinces, escalated appeals,
  Kashif-block alerts, port requests
- Port-request fulfilment via iptables on Sultan approval
- AppRole authentication to OpenBao with least-privilege policy
- Sultan-unsolicited alerts when Kashif block counters exceed
  thresholds or lease renewals fail

**Deferred:**
- Content inspection of its own (Kashif does it)
- Complex dashboard mutations (dashboard stays read-only)
- SentinelGate-style session tracking
- Auto-unseal of OpenBao (manual unseal in Phase 1)
- AppRole secret_id rotation on Aga restart (deferred with
  compromised-core hardening)
- Dual audit sinks / signed audit records
- Multi-operator Sultan support
