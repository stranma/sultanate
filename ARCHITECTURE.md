# Sultanate Architecture

## 1. System Overview

```
                          +-------------+
                          |   Sultan    |
                          | (human, TG) |
                          +------+------+
                                 |
                    Telegram API |
                +----------------+----------------+
                |                                 |
        +-------v--------+              +---------v-------+
        |    Vizier      |              |      Aga        |
        | (OpenClaw)     |              |   (OpenClaw)    |
        | vizier user    |              | root            |
        +---+----+---+---+              +---+----+--------+
            |    |   |                      |    |
  Docker    |    |   | Divan API            |    | Divan API
  socket    |    |   |                      |    |
   +--------+   |   |    +-----------+      |    |
   |             |   +--->|           |<-----+    |
   |             |        |   Divan   |           |
   |   Telegram  |        | SQLite +  |           | OpenBao
   |   API       |        | HTTP API  |           | API (local)
   |             |        | port 8600 |           |
   |             |        | dashboard |           |
   |             |        | port 8601 |           |
   |             |        +-----+-----+           |
   |             |              |                  |
   |             |              | polls            |
   |             |              |                  |
   |        +----v--------------v-------+          |
   |        |     Janissary + Kashif    |          |
   |        |  mitmproxy + appeal +     |          |
   |        |  WireGuard server +       |          |
   |        |  local LLM content screen |          |
   |        |  ports: 8080/8081/8082    |          |
   |        +----+-------------+--------+          |
   |             |             |                   |
   |   WireGuard |             | internet          |
   |   tunnels   |             | egress            |
   |             |             |                   |
   |    +--------v--------+    |    +--------------+
   |    |   wg-client     |    |    |
   |    |   sidecars      |    +--->| Internet
   |    +--------+--------+         +--------------+
   |             |
   |    shared   |
   |    network  |
   |    namespace|
   |             |
   +----+--------v--------+
        |    Provinces     |
        | (Docker, OpenClaw)|
        | untrusted        |
        | no direct egress |
        +---------+--------+
                  |
                  | appeal API
                  | (HTTP, port 8081)
                  v
            Janissary appeal endpoint
            (appeals screened by Kashif)

Trust levels:
  Trusted:      Aga (root, host access, secrets),
                Janissary (root, network enforcement),
                Kashif (runs alongside Janissary, local LLM content screener)
  Semi-trusted: Vizier (Docker group, no secrets)
  Untrusted:    Provinces (sandboxed, no internet, no secrets)
```

## 2. Network / Traffic Flow

### Outbound request (province -> internet)

```
Province container                wg-client sidecar
+------------------+             +------------------+
| Agent sends      |  shared     | iptables NAT     |
| POST https://    |  network    | redirects 80/443 |
| api.github.com   +----------->| to WireGuard      |
| (no auth header) |  namespace  | tunnel            |
+------------------+             +--------+---------+
                                          |
                                 WireGuard tunnel
                                 (10.13.13.x -> 10.13.13.1)
                                          |
                                 +--------v---------+
                                 | Janissary        |
                                 | mitmproxy:8080   |
                                 +--------+---------+
                                          |
                              +-----------v-----------+
                              | 1. MITM decrypt       |
                              |    (Sultanate CA)     |
                              |                       |
                              | 2. Identify source    |
                              |    (WireGuard peer IP)|
                              |                       |
                              | 3. Evaluate rules:    |
                              |    Blacklisted?  BLOCK|
                              |    Whitelisted?  PASS |
                              |    GET/HEAD?     PASS |
                              |    POST/PUT/etc? BLOCK|
                              |    Approved?     PASS |
                              |                       |
                              | 4. Inject credentials |
                              |    (if grant exists;  |
                              |    OpenBao lease must |
                              |    still be valid)    |
                              |                       |
                              | 5. Log to audit.jsonl |
                              +-----------+-----------+
                                          |
                                          | re-encrypted
                                          v
                                    +-----------+
                                    | Internet  |
                                    | (upstream)|
                                    +-----------+
```

### Appeal flow (blocked write request)

```
Province      Janissary     Kashif         Divan        Vizier     Sultan
   |             |             |             |            |          |
   |-- POST ---->|             |             |            |          |
   |             |-- BLOCK     |             |            |          |
   |<-- 403 + URL|             |             |            |          |
   |             |             |             |            |          |
   |-- appeal -->|             |             |            |          |
   |             |-- screen -->|             |            |          |
   |             |             |-- regex     |            |          |
   |             |             |-- classifier|            |          |
   |             |             |-- LLM judge |            |          |
   |             |             |             |            |          |
   |             |  obvious-safe: approve    |            |          |
   |             |  obvious-bad:  block      |            |          |
   |             |  unclear:      escalate   |            |          |
   |             |             |             |            |          |
   |             |<-- verdict -|             |            |          |
   |             |-- POST /appeals --------->|            |          |
   |<-- pending -|             |             |            |          |
   |             |             |             |<-- poll ---|          |
   |             |             |             |-- appeal ->|          |
   |             |             |             |            |-- TG --->|
   |             |             |             |            |          |
   |             |             |             |            |<-- OK ---|
   |             |             |             |<-- PATCH --|          |
   |             |<-- poll (5s)-             |             |          |
   |             |   approved  |             |            |          |
   |-- retry --->|             |             |            |          |
   |             |-- PASS      |             |            |          |
   |<-- 200 -----|             |             |            |          |
```

### Appeal flow -- what happens in what order

Setup: province `prov-a1b2c3` wants to POST source code to a non-
whitelisted, non-blacklisted service.

```
T+0.0s   Pasha (inside province):
         POST https://<new-service>/upload  (500 bytes)

T+0.1s   Janissary: source_ip 10.13.13.5 -> rule 4 (non-whitelist
         write) -> BLOCK. Returns 403 + appeal_url to Pasha.

T+0.2s   Pasha: calls appeal_request(url, method, payload,
         justification) via the Janissary security MCP.

T+0.3s   Janissary does two writes in parallel:
         (a) POST /appeals to Divan -> creates appeal-m1n2o3 with
             status=pending, kashif_verdict=null
         (b) POST /screen/appeal to Kashif -> payload + justification
         Returns 202 Accepted to Pasha.

T+0.4s   Kashif Layer 1 (LLM Guard regex, ~10 ms) -> pass
T+0.6s   Kashif Layer 2 (Prompt Guard 2 22M, ~200 ms) -> pass
T+2.0s   Kashif Layer 3 (Llama Guard 3 1B Q4, ~1-2 s) -> verdict
```

**Three possible outcomes at T+2.0s:**

```
 CASE A: Kashif = "allow" (obvious safe)
---------------------------------------------------
T+2.1s   Kashif: PATCH /appeals/m1n2o3/kashif_verdict
                  { kashif_verdict: "allow", ... }
T+2.2s   Divan atomically transitions:
                  status         := "approved"
                  decision       := "one-time"
                  resolved_at    := now
         Audit entry written with severity=info.
         Sultan NOT notified (obvious safe). Aga NOT notified.
T+next   Pasha retries -> Janissary sees approved in 5-minute
         window -> PASS.

 CASE B: Kashif = "block" (obvious bad)
---------------------------------------------------
T+2.1s   Kashif: PATCH with kashif_verdict="block".
T+2.2s   Divan atomically transitions:
                  status         := "denied"
                  resolved_at    := now
         Audit entry written with severity=ALERT.
         Decision is final, but...
T+5s     Vizier polls /audit?severity=alert&since=<last>
         -> sends INFORMATIONAL Telegram to Sultan:
           "Prov backend-refactor appeal AUTO-BLOCKED by Kashif.
            Target: <url>. Justification: '<text>'.
            Kashif notes: <...>. Decision is final; review if
            pattern repeats."
T+5s     Aga polls the same audit feed. Tracks a counter per
         province of Kashif blocks; if threshold exceeded,
         alerts Sultan with a recommendation ("Prov backend-
         refactor has 3 Kashif blocks in 10 min. Consider
         destroying the province.").
T+next   Pasha retries -> still rule-4-blocked, no approval
         record -> 403 again.

 CASE C: Kashif = "escalate" (unsure) or Kashif TIMEOUT
---------------------------------------------------
T+2.1s   Kashif: PATCH kashif_verdict="escalate" (or Janissary
         itself writes "escalate" after its configured timeout).
T+2.2s   Divan leaves status=pending.
T+5-10s  Vizier polls /appeals?status=pending&kashif_verdict=escalate
         -> sends ACTIONABLE Telegram to Sultan:
           "Prov backend-refactor appeal NEEDS DECISION.
            Target: <url>. Justification: '<text>'. Kashif:
            escalate (notes: <...>).
            Approve once / approve forever / deny / kill province?"
         Aga polls same records; may add commentary to Sultan
         ("This looks like a normal test-upload pattern" or
          "I've seen prov-a1b2c3 hit 3 pastebin-ish domains in
          the last hour -- suspicious").
T+human  Sultan decides. Vizier writes decision to Divan.
T+next   Pasha retries -> PASS or 403 per decision.
```

**Why both Sultan and Aga see Case B:** Kashif=block handles the
request correctly, but a streak of blocks is a signal that the Pasha
may be going off the rails. Sultan + Aga see these events so they can
assess behaviour (add the domain to blacklist, destroy the province,
ask Kashif to tighten screening, etc.). The goal isn't to re-decide
Kashif's verdict -- it's to catch drift.

**Why Kashif fails closed:** if Kashif's LLM is down or times out,
Janissary writes kashif_verdict="escalate" and the appeal flows to
Sultan exactly like Case C. Kashif never auto-approves anything in
the degraded path.

### Mid-task credential request flow

Distinct from the appeal flow. An appeal is "this specific request
was blocked, let me through this once". A credential request is
"please provision a new long-lived credential so I can access service
X going forward." Examples: a Pasha needs API access to a new SaaS
service it didn't have at province creation; the GitHub App
installation needs to grow to cover a new repo.

```
Pasha             Janissary    Kashif        Divan      Vizier      Aga         Sultan
  |                  |            |             |           |          |            |
  |-- request_access(service, scope, justification) MCP call           |            |
  |--POST /api/req->|            |             |           |          |            |
  |                 |--screen/ingress(text)-->|             |          |            |
  |                 |            |--regex     |             |          |            |
  |                 |            |--PromptGuard               |        |            |
  |                 |            |--LlamaGuard                |        |            |
  |                 |<--verdict--|             |           |          |            |
  |                 |--POST /access_request -->|           |          |            |
  |<-- pending -----|            |             |           |          |            |
```

**Three outcomes at the verdict step (~T+2.0s):**

```
 CASE A: Kashif = "allow"
---------------------------------------------------
T+2.2s   Divan stores the access_request with status=pending,
         kashif_verdict=allow. Audit severity = info.
T+5-10s  Vizier polls /access_requests?status=pending and sends
         actionable Telegram to Sultan:
           "Prov backend-refactor wants access to api.acme.com
            (write scope). Justification: '<text>'.
            Kashif: allow (notes: '...').
            Approve / deny?"
T+5-10s  Aga also polls. Adds advisory context to Sultan via its
         own Telegram channel: "I've seen prov-a1b2c3 use this
         class of API before in test mode -- this is consistent
         with the task you assigned."
T+human  Sultan replies "approve" to Vizier.
         Vizier writes status=approved to Divan.
T+poll   Aga sees the approved access_request:
           - For dynamic engines (DB creds, GitHub App scope
             expansion, etc.): Aga calls OpenBao to mint, receives
             token + lease, writes lease-bound grant to Divan.
           - For KV fallback (Sultan-pasted): Aga prompts Sultan
             via Telegram for the token, stores in OpenBao KV,
             writes grant with null lease.
T+5s     Janissary picks up the new grant on its next poll;
         Pasha's future requests to the new service get
         credential injected.

 CASE B: Kashif = "block"
---------------------------------------------------
T+2.2s   Divan stores access_request with status=denied,
         kashif_verdict=block. Audit severity = ALERT.
T+5s     Vizier sends INFORMATIONAL Telegram to Sultan:
           "Prov backend-refactor's access request was
            AUTO-BLOCKED by Kashif. Service: api.suspicious.xyz.
            Justification: '<text>'. Kashif notes: '<...>'.
            Decision is final; review if pattern repeats."
T+5s     Aga polls Kashif-block audit; increments per-province
         counter. If >= 3 blocks in 10 min, sends recommendation
         to Sultan ("consider destroying prov-a1b2c3").
T+next   Pasha's next request to api.suspicious.xyz still gets
         403 from Janissary -- no grant, no whitelist.

 CASE C: Kashif = "escalate" (or Kashif TIMEOUT)
---------------------------------------------------
T+2.2s   Divan stores access_request with status=pending,
         kashif_verdict=escalate.
T+5-10s  Vizier sends actionable Telegram (same as Case A) but
         with escalate framing:
           "Prov backend-refactor wants access to <service>.
            Kashif: escalate (notes: '...'). NEEDS DECISION.
            Approve / deny / kill province?"
T+5-10s  Aga adds context, possibly with a recommendation
         (e.g., "I'd lean deny -- this is the third unfamiliar
         service this province has asked for in an hour").
T+human  Sultan decides. Approve path follows Case A from the
         "Aga sees the approved access_request" step onward.
```

**Difference from the appeal flow:** the appeal flow's allow path is
auto-resolved by Divan (no Sultan involvement). The credential-request
flow's allow path **always** involves Sultan, because granting a
long-lived credential is more consequential than letting a single
write through. Kashif's allow verdict only means "this text isn't
malicious" -- it does not mean "Sultan would approve this token."

### Kill-switch (WireGuard down)

```
Province                wg-client sidecar
   |                       |
   |-- any request ------->|
   |                       |-- WireGuard tunnel DOWN
   |                       |-- iptables kill-switch:
   |                       |   all OUTPUT except wg0 -> DROP
   |                       |
   |<-- connection timeout-|
   |                       |
   (no traffic leaks to internet)
```

## 3. Province Lifecycle

### Creation

```
Sultan (Telegram)
   |
   | "Set up a coding agent for the EFM repo"
   v
Vizier (OpenClaw agent, understands natural language)
   |
   | Vizier interprets the request, picks the right firman
   | and berat, then runs CLI commands via bash tool:
   |   vizier-cli create openclaw-firman --berat openclaw-coding-berat --repo stranma/EFM
   |
   |  1. Load firman YAML + berat YAML
   |  2. Generate WireGuard peer config (assign 10.13.13.N)
   |  3. Generate province ID (prov-XXXXXX)
   |
   |  4. docker create wg-client-prov-XXXXXX
   |     (cap_add: NET_ADMIN, WireGuard conf mounted)
   |
   |  5. docker create sultanate-{name}
   |     (network_mode: container:wg-client-prov-XXXXXX)
   |     (CA cert mounted, host volume mounted)
   |
   |  6. Register in Divan: POST /provinces
   |     {id, name, ip: 10.13.13.N, status: creating}
   |
   |  7. Post berat whitelist: PUT /whitelists/{id}
   |  8. Post berat port requests: POST /port_requests
   |
   v
Aga (watches Divan)
   |
   |  9. Sees new province
   | 10. Ask Sultan for GitHub token via Telegram
   |     Sultan provides token
   | 11. Store in OpenBao, receive lease ID
   |     Write grant to Divan: POST /grants
   |     {source_ip, domain, inject, openbao_lease_id,
   |      lease_expires_at}
   |
   v
Vizier (continues)
   |
   | 12. docker start wg-client-prov-XXXXXX
   | 13. docker start sultanate-{name}
   | 14. docker cp CA cert + update-ca-certificates
   | 15. docker exec: clone repo (through Janissary)
   | 16. docker exec: apply berat (SOUL.md, AGENTS.md,
   |                               ~/.openclaw/openclaw.json)
   | 17. docker exec: openclaw gateway --port 18789
   |
   | 18. Update Divan: PATCH /provinces/{id} {status: running}
   |
   v
Province is live, agent connects to Sultan via Telegram
```

### Stop / Destroy

```
Sultan: "Stop the EFM agent"         Sultan: "Destroy the EFM province"
   |                                    |
   v                                    v
Vizier (runs vizier-cli stop)        Vizier (runs vizier-cli destroy)
   |                                    |
   | 1. docker stop sultanate-{name}    | 1. docker stop sultanate-{name}
   | 2. docker stop wg-client-prov-XX   | 2. docker stop wg-client-prov-XX
   | 3. PATCH /provinces/{id}           | 3. docker rm sultanate-{name}
   |    {status: stopped}               | 4. docker rm wg-client-prov-XX
   |                                    | 5. Remove WireGuard peer from
   | (can restart later)                |    Janissary server config
   |                                    | 6. DELETE /provinces/{id}
                                        | 7. Aga revokes all OpenBao
                                        |    leases for this province;
                                        |    any it misses expires by
                                        |    TTL server-side anyway
                                        | 8. Cleanup host volume
                                        |    (or preserve for inspection)
```

---

## User Stories

Each story describes a user-visible scenario, the components involved, and
assertions that map to integration/e2e test checks.

---

### US-1: Deploy Agent

**Actor:** Sultan (via Telegram)
**Action:** Tell Vizier in plain language to set up an agent for a repo

```
Sultan: "Set up a coding agent for the EFM repo"
Vizier interprets, picks firman + berat, runs: vizier-cli create openclaw-firman --berat openclaw-coding-berat --repo stranma/EFM
```

**Expected outcome:** A running province with the agent connected to Telegram,
workspace cloned, and able to receive instructions.

**Components:** Vizier, Divan, Aga, Janissary, Province, wg-client, OpenBao

**Test assertions:**
- [ ] Province container is running (`docker ps`)
- [ ] wg-client sidecar is running
- [ ] Divan has province record with status=running and assigned IP
- [ ] Divan has at least one grant for the province (GitHub token) with valid `openbao_lease_id`
- [ ] Province can reach api.github.com through Janissary (curl from inside)
- [ ] Repo is cloned into /opt/data/workspace
- [ ] SOUL.md, AGENTS.md, and ~/.openclaw/openclaw.json written from berat templates
- [ ] Agent process is running (`openclaw gateway` on port 18789)
- [ ] WireGuard tunnel is established (ping 10.13.13.1 from province)

---

### US-2: Agent Reads From Internet

**Actor:** Province agent (automated)
**Action:** HTTP GET to a non-whitelisted domain (e.g., docs.python.org)

**Expected outcome:** Request passes. Agent can browse, read documentation,
download packages from any non-blacklisted domain.

**Components:** Province, wg-client, Janissary

**Test assertions:**
- [ ] GET https://docs.python.org returns 200
- [ ] GET http://example.com returns 200
- [ ] Janissary audit log shows decision=allow, rule=read_only
- [ ] No credential injection occurred (mitm=true, credential_injected=false)

---

### US-3: Agent Pushes to GitHub (Credential Injection)

**Actor:** Province agent (automated)
**Action:** POST to api.github.com (whitelisted domain with grant)

**Expected outcome:** Janissary injects the GitHub token into the Authorization
header. GitHub accepts the request. Agent never sees the token.

**Components:** Province, wg-client, Janissary, Divan (grant), OpenBao (lease)

**Test assertions:**
- [ ] POST https://api.github.com/repos/{owner}/{repo}/pulls returns 201
- [ ] Janissary audit log shows credential_injected=true
- [ ] Province container has no GitHub token in its environment (`env | grep -i github` empty)
- [ ] Province container has no GitHub token in git config
- [ ] Divan grant record exists for (province_ip, api.github.com) with valid OpenBao lease
- [ ] If lease expired (or revoked in OpenBao), next injection is skipped and request fails closed

---

### US-4: Agent Blocked by Write-Block, Appeals, Gets Approved

**Actor:** Province agent (automated), Kashif (automated screen), then Aga + Sultan (if unclear)
**Action:** POST to a non-whitelisted, non-blacklisted domain

**Expected outcome:** First request blocked (403). Agent appeals. Kashif
screens the payload. If obvious-safe or obvious-bad, auto-decided. Otherwise,
Aga and Sultan review. Agent retries within timeout window. Request passes
(or stays blocked).

**Components:** Province, Janissary, Kashif, Divan, Aga, Vizier, Sultan (Telegram)

**Test assertions:**
- [ ] First POST returns 403 with appeal URL in body
- [ ] POST /api/appeal returns {status: pending}
- [ ] Divan has appeal record with status=pending
- [ ] Janissary forwards appeal payload to Kashif `/screen/appeal`
- [ ] Kashif returns one of {allow, block, escalate} within 5s (fail-closed if timeout)
- [ ] If Kashif=allow: Divan appeal auto-transitions to approved
- [ ] If Kashif=block: Divan appeal auto-transitions to denied
- [ ] If Kashif=escalate: Vizier relays to Sultan via Telegram (as before)
- [ ] After Sultan approves: Divan appeal status=approved
- [ ] Janissary picks up approval within 5s (poll interval)
- [ ] Retry POST to same URL returns 200 (or upstream status)
- [ ] Janissary audit log shows decision=allow, rule=approved_appeal, kashif_verdict recorded
- [ ] After timeout (configurable, default 5 min): same POST returns 403 again

---

### US-5: Agent Hits Blacklist

**Actor:** Province agent (automated)
**Action:** Any request (GET or POST) to a blacklisted domain

**Expected outcome:** Request blocked. No appeal option for blacklisted domains.

**Components:** Province, wg-client, Janissary, Divan (blacklist)

**Test assertions:**
- [ ] GET https://pastebin.com returns 403
- [ ] POST https://pastebin.com returns 403
- [ ] Response body indicates blacklist (not write-block)
- [ ] Janissary audit log shows decision=block_blacklist
- [ ] Domain is in Divan's global blacklist
- [ ] Appeal for a blacklisted domain is rejected or has no effect

---

### US-6: Agent Requests New Access (Credentials for New Service)

**Actor:** Province agent (automated), Kashif (automated screen), Sultan, Aga
**Action:** Agent calls request_access for a service it doesn't have credentials for

**Expected outcome:** Request payload (service + scope + justification) is
screened by Kashif before reaching Aga's LLM context. Cleared requests flow
to Sultan. Sultan tells Aga to provision. Aga issues the credential via
OpenBao (gets a lease), writes grant to Divan. Agent can now use the service.

**Components:** Province, Janissary (appeal API), Kashif, Divan, Vizier, Sultan, Aga, OpenBao

**Test assertions:**
- [ ] POST /api/request_access returns {status: pending}
- [ ] Janissary forwards the request text to Kashif `/screen/ingress`
- [ ] Kashif returns allow/block/escalate within 5s (fail-closed if timeout)
- [ ] If Kashif=block: access request auto-denied, alert to Sultan
- [ ] Otherwise: Divan has access request record, Vizier relays to Sultan
- [ ] Sultan instructs Aga to provision credential
- [ ] Aga calls OpenBao to create/retrieve the credential; receives value + lease
- [ ] Aga writes grant to Divan (POST /grants with openbao_lease_id)
- [ ] Janissary picks up new grant within 5s
- [ ] Agent's next request to the new service gets credential injected
- [ ] Janissary audit log shows credential_injected=true for new domain

---

### US-7: Sultan Adds Permanent Whitelist Entry

**Actor:** Sultan (via Aga)
**Action:** Sultan tells Aga to add a domain to a province's whitelist

**Expected outcome:** Domain added to whitelist in Divan. Janissary picks up
the change. All methods (GET, POST, PUT, DELETE) now pass for that domain.

**Components:** Sultan, Aga, Divan, Janissary

**Test assertions:**
- [ ] Before: POST to target domain returns 403 (write-block)
- [ ] Aga writes to Divan: PUT /whitelists/{province_id} (adds domain)
- [ ] After Janissary polls (<=5s): POST to target domain returns 200
- [ ] GET to target domain still works
- [ ] Janissary audit log shows rule=whitelist (not read_only or approved_appeal)

---

### US-8: Sultan Opens a Non-HTTP Port

**Actor:** Sultan (via Aga)
**Action:** Agent needs database access (e.g., PostgreSQL on port 5432)

**Expected outcome:** Berat declares the port. Vizier writes request to Divan.
Aga asks Sultan for approval. On approval, Aga opens the host:port pair via
iptables/Docker rules.

**Components:** Province, Vizier, Divan, Aga, Sultan

**Test assertions:**
- [ ] Berat has port_declarations entry for host:5432
- [ ] Divan has port_request record (status=pending)
- [ ] Aga relays request to Sultan via Telegram
- [ ] After Sultan approves: Aga opens iptables rule
- [ ] Province can connect to target host on port 5432
- [ ] Divan port_request updated to status=approved
- [ ] Without approval: connection to host:5432 times out (kill-switch)

---

### US-9: Province Stop and Destroy

**Actor:** Sultan (via Telegram)
**Action:** Tell Vizier to stop a province, then destroy it

**Expected outcome:** Province and sidecar containers stopped/removed. Divan
records updated. WireGuard peer cleaned up. OpenBao leases revoked by Aga
(or expire on TTL if Aga misses them).

**Components:** Sultan, Vizier, Province, wg-client, Divan, Janissary (WireGuard), Aga, OpenBao

**Test assertions (stop):**
- [ ] Province container status: exited
- [ ] wg-client sidecar status: exited
- [ ] Divan province status=stopped
- [ ] WireGuard tunnel down (no ping from Janissary to province IP)
- [ ] Province data preserved on host (/opt/sultanate/provinces/{id}/data exists)

**Test assertions (destroy):**
- [ ] Province container removed (`docker ps -a` doesn't show it)
- [ ] wg-client container removed
- [ ] Divan province record deleted (or status=destroyed)
- [ ] WireGuard peer config removed from Janissary
- [ ] Grants for this province's IP removed from Divan
- [ ] Whitelist for this province removed from Divan
- [ ] Aga revoked all OpenBao leases for this province (or TTL will clear them server-side)

---

### US-10: System Startup (Boot Order)

**Actor:** System (deploy script)
**Action:** Start all components from scratch

**Expected outcome:** Components start in order, health checks pass, system
is ready to create provinces.

**Components:** OpenBao, Divan, Janissary, Kashif, Aga, Vizier

**Test assertions:**
- [ ] OpenBao starts first (manual unseal by Sultan); /v1/sys/health returns 200
- [ ] Divan starts, /health returns 200
- [ ] Janissary starts, waits for Divan health, then /health returns 200
- [ ] Janissary in fail-closed mode until first Divan poll succeeds
- [ ] Before first poll: any traffic through Janissary returns 503
- [ ] After first poll: traffic rules applied normally
- [ ] Kashif starts, loads all three screener layers (LLM Guard regex, Prompt Guard 2 22M, Llama Guard 3 1B Q4), /health returns 200; all three models resident
- [ ] Aga starts (host networking, not through Janissary); authenticates to OpenBao via AppRole
- [ ] Vizier starts after Janissary + Kashif healthy
- [ ] Vizier's DivanClient.wait_for_divan() succeeds
- [ ] System ready: `vizier-cli create` command works
- [ ] WireGuard server interface is up on Janissary (10.13.13.1)
- [ ] Divan dashboard reachable at `http://<host-tailscale-ip>:8601` from any device on Sultan's tailnet (or via SSH tunnel in fallback mode)
