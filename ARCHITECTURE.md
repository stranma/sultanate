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
        |    Vizier      |              |    Sentinel     |
        | (Hermes agent) |              | (Hermes agent)  |
        | vizier user    |              | root            |
        +---+----+---+---+              +---+----+--------+
            |    |   |                      |    |
  Docker    |    |   | Divan API            |    | Divan API
  socket    |    |   |                      |    |
   +--------+   |   |    +-----------+      |    |
   |             |   +--->|           |<-----+    |
   |             |        |   Divan   |           |
   |   Telegram  |        | SQLite +  |           | Infisical
   |   API       |        | HTTP API  |           | API
   |             |        | port 8600 |           |
   |             |        +-----+-----+           |
   |             |              |                  |
   |             |              | polls            |
   |             |              |                  |
   |        +----v--------------v---+              |
   |        |      Janissary        |              |
   |        |  mitmproxy + appeal   |              |
   |        |  WireGuard server     |              |
   |        |  ports: 8080/8081     |              |
   |        +----+-------------+----+              |
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
        | (Docker, Hermes) |
        | untrusted        |
        | no direct egress |
        +---------+--------+
                  |
                  | appeal API
                  | (HTTP, port 8081)
                  v
            Janissary appeal endpoint

Trust levels:
  Trusted:      Sentinel (root, host access, secrets)
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
                              |    (if grant exists)  |
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
Province                Janissary              Divan          Vizier         Sultan
   |                       |                     |              |              |
   |-- POST example.com -->|                     |              |              |
   |                       |-- rule 4: BLOCK     |              |              |
   |<-- 403 + appeal URL --|                     |              |              |
   |                       |                     |              |              |
   |-- POST /api/appeal -->|                     |              |              |
   |   (port 8081)         |-- POST /appeals --->|              |              |
   |<-- {status: pending} -|                     |              |              |
   |                       |                     |              |              |
   |                       |                     |<-- poll -----|              |
   |                       |                     |-- appeal --->|              |
   |                       |                     |              |-- TG msg --->|
   |                       |                     |              |              |
   |                       |                     |              |<-- approve --|
   |                       |                     |<-- PATCH ----|              |
   |                       |                     |  (approved)  |              |
   |                       |                     |              |              |
   |                       |<-- poll (5s) -------|              |              |
   |                       |   approved_appeals  |              |              |
   |                       |                     |              |              |
   |-- POST example.com -->|                     |              |              |
   |                       |-- approved appeal   |              |              |
   |                       |   PASS              |              |              |
   |<-- 200 OK ------------|                     |              |              |
```

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
Vizier (Hermes agent, understands natural language)
   |
   | Vizier interprets the request, picks the right firman
   | and berat, then runs CLI commands via terminal tool:
   |   vizier-cli create hermes-firman --berat hermes-coding-berat --repo stranma/EFM
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
Sentinel (watches Divan)
   |
   |  9. Sees new province
   | 10. Ask Sultan for GitHub token via Telegram
   |     Sultan provides token
   | 11. Store in Infisical, write grant to Divan
   |     POST /grants {source_ip, domain, inject}
   |
   v
Vizier (continues)
   |
   | 12. docker start wg-client-prov-XXXXXX
   | 13. docker start sultanate-{name}
   | 14. docker cp CA cert + update-ca-certificates
   | 15. docker exec: clone repo (through Janissary)
   | 16. docker exec: apply berat (SOUL.md, AGENTS.md, config.yaml)
   | 17. docker exec: hermes gateway (agent starts)
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
                                        | 7. Cleanup host volume
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
Vizier interprets, picks firman + berat, runs: vizier-cli create hermes-firman --berat hermes-coding-berat --repo stranma/EFM
```

**Expected outcome:** A running province with the agent connected to Telegram,
workspace cloned, and able to receive instructions.

**Components:** Vizier, Divan, Sentinel, Janissary, Province, wg-client

**Test assertions:**
- [ ] Province container is running (`docker ps`)
- [ ] wg-client sidecar is running
- [ ] Divan has province record with status=running and assigned IP
- [ ] Divan has at least one grant for the province (GitHub token)
- [ ] Province can reach api.github.com through Janissary (curl from inside)
- [ ] Repo is cloned into /opt/data/workspace
- [ ] SOUL.md and config.yaml written from berat templates
- [ ] Agent process is running (hermes gateway)
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

**Components:** Province, wg-client, Janissary, Divan (grant)

**Test assertions:**
- [ ] POST https://api.github.com/repos/{owner}/{repo}/pulls returns 201
- [ ] Janissary audit log shows credential_injected=true
- [ ] Province container has no GitHub token in its environment (`env | grep -i github` empty)
- [ ] Province container has no GitHub token in git config
- [ ] Divan grant record exists for (province_ip, api.github.com)

---

### US-4: Agent Blocked by Write-Block, Appeals, Gets Approved

**Actor:** Province agent (automated), then Sultan (manual approval)
**Action:** POST to a non-whitelisted, non-blacklisted domain

**Expected outcome:** First request blocked (403). Agent appeals. Sultan
approves. Agent retries within timeout window. Request passes.

**Components:** Province, Janissary, Divan, Vizier, Sultan (Telegram)

**Test assertions:**
- [ ] First POST returns 403 with appeal URL in body
- [ ] POST /api/appeal returns {status: pending}
- [ ] Divan has appeal record with status=pending
- [ ] Vizier relays appeal to Sultan via Telegram
- [ ] After Sultan approves: Divan appeal status=approved
- [ ] Janissary picks up approval within 5s (poll interval)
- [ ] Retry POST to same URL returns 200 (or upstream status)
- [ ] Janissary audit log shows decision=allow, rule=approved_appeal
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

**Actor:** Province agent (automated), then Sultan, then Sentinel
**Action:** Agent calls request_access for a service it doesn't have credentials for

**Expected outcome:** Request flows to Sultan. Sultan tells Sentinel to provision
the credential. Sentinel stores in Infisical, writes grant to Divan. Agent can
now use the service.

**Components:** Province, Janissary (appeal API), Divan, Vizier, Sultan, Sentinel, Infisical

**Test assertions:**
- [ ] POST /api/request_access returns {status: pending}
- [ ] Divan has access request record
- [ ] Vizier relays to Sultan via Telegram
- [ ] Sultan instructs Sentinel to provision token
- [ ] Sentinel writes secret to Infisical
- [ ] Sentinel writes grant to Divan (POST /grants)
- [ ] Janissary picks up new grant within 5s
- [ ] Agent's next request to the new service gets credential injected
- [ ] Janissary audit log shows credential_injected=true for new domain

---

### US-7: Sultan Adds Permanent Whitelist Entry

**Actor:** Sultan (via Sentinel)
**Action:** Sultan tells Sentinel to add a domain to a province's whitelist

**Expected outcome:** Domain added to whitelist in Divan. Janissary picks up
the change. All methods (GET, POST, PUT, DELETE) now pass for that domain.

**Components:** Sultan, Sentinel, Divan, Janissary

**Test assertions:**
- [ ] Before: POST to target domain returns 403 (write-block)
- [ ] Sentinel writes to Divan: PUT /whitelists/{province_id} (adds domain)
- [ ] After Janissary polls (<=5s): POST to target domain returns 200
- [ ] GET to target domain still works
- [ ] Janissary audit log shows rule=whitelist (not read_only or approved_appeal)

---

### US-8: Sultan Opens a Non-HTTP Port

**Actor:** Sultan (via Sentinel)
**Action:** Agent needs database access (e.g., PostgreSQL on port 5432)

**Expected outcome:** Berat declares the port. Vizier writes request to Divan.
Sentinel asks Sultan for approval. On approval, Sentinel opens the
host:port pair via iptables/Docker rules.

**Components:** Province, Vizier, Divan, Sentinel, Sultan

**Test assertions:**
- [ ] Berat has port_declarations entry for host:5432
- [ ] Divan has port_request record (status=pending)
- [ ] Sentinel relays request to Sultan via Telegram
- [ ] After Sultan approves: Sentinel opens iptables rule
- [ ] Province can connect to target host on port 5432
- [ ] Divan port_request updated to status=approved
- [ ] Without approval: connection to host:5432 times out (kill-switch)

---

### US-9: Province Stop and Destroy

**Actor:** Sultan (via Telegram)
**Action:** Tell Vizier to stop a province, then destroy it

**Expected outcome:** Province and sidecar containers stopped/removed. Divan
records updated. WireGuard peer cleaned up.

**Components:** Sultan, Vizier, Province, wg-client, Divan, Janissary (WireGuard)

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

---

### US-10: System Startup (Boot Order)

**Actor:** System (deploy script)
**Action:** Start all components from scratch

**Expected outcome:** Components start in order, health checks pass, system
is ready to create provinces.

**Components:** Divan, Janissary, Sentinel, Vizier

**Test assertions:**
- [ ] Divan starts first, /health returns 200
- [ ] Janissary starts, waits for Divan health, then /health returns 200
- [ ] Janissary in fail-closed mode until first Divan poll succeeds
- [ ] Before first poll: any traffic through Janissary returns 503
- [ ] After first poll: traffic rules applied normally
- [ ] Sentinel starts (host networking, not through Janissary)
- [ ] Vizier starts after Janissary healthy
- [ ] Vizier's DivanClient.wait_for_divan() succeeds
- [ ] System ready: `vizier-cli create` command works
- [ ] WireGuard server interface is up on Janissary (10.13.13.1)
