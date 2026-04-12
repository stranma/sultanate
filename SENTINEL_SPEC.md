# Sentinel Technical Specification

> Implementation spec for Sentinel, the secret management agent.
> For product requirements see [SENTINEL_MVP_PRD.md](SENTINEL_MVP_PRD.md).
> For shared state API see [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md).
> For architecture context see [SULTANATE_MVP.md](SULTANATE_MVP.md).

## 1. Infisical Integration

Sentinel uses the Infisical CLI (`infisical`) via the Hermes terminal tool.
Infisical runs locally on the host. Sentinel accesses it directly (no proxy).

### Project Structure

- **Project:** `sultanate`
- **Environments:** One per province. Environment name = province ID (e.g., `prov-a1b2c3`).
- **Secret naming:** `<SERVICE>_<SCOPE>` (e.g., `GITHUB_TOKEN`, `NPM_TOKEN`).

### Authentication

Sentinel authenticates via Machine Identity (Universal Auth). Credentials
stored in `/opt/sultanate/sentinel.env`:

```bash
INFISICAL_CLIENT_ID=mi.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
INFISICAL_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
INFISICAL_API_URL=http://127.0.0.1:8070/api
```

The Infisical CLI reads these from environment. Sentinel's HERMES_HOME
startup script sources this file before running any `infisical` command.

### CLI Commands

**Login (session token, run once at startup):**
```bash
export INFISICAL_TOKEN=$(infisical login --method=universal-auth \
  --client-id="$INFISICAL_CLIENT_ID" \
  --client-secret="$INFISICAL_CLIENT_SECRET" \
  --plain --silent)
```

**Create environment for a new province:**
```bash
infisical secrets set --projectId sultanate --env prov-a1b2c3 \
  GITHUB_TOKEN="EXAMPLE_GITHUB_TOKEN_HERE"
```

Infisical auto-creates the environment on first secret write if it doesn't
exist. No separate environment-creation step needed.

**Read a secret:**
```bash
infisical secrets get GITHUB_TOKEN \
  --projectId sultanate --env prov-a1b2c3 --plain
```

**List all secrets in a province environment:**
```bash
infisical secrets list --projectId sultanate --env prov-a1b2c3
```

**Delete a single secret:**
```bash
infisical secrets delete GITHUB_TOKEN \
  --projectId sultanate --env prov-a1b2c3
```

**Delete all secrets for a province (on revocation):**
```bash
for secret in $(infisical secrets list --projectId sultanate --env prov-a1b2c3 --plain --silent | awk '{print $1}'); do
  infisical secrets delete "$secret" --projectId sultanate --env prov-a1b2c3
done
```

All commands include `--silent` in production to suppress informational output.

## 2. Grant Provisioning Workflow

Triggered when Sentinel's polling loop detects a new province with
`status=creating` in Divan.

### Step-by-Step

**Step 1 -- Read province record from Divan:**

```
GET http://127.0.0.1:8600/provinces/{id}
Authorization: Bearer <DIVAN_KEY_SENTINEL>
```

Response (relevant fields):
```json
{
  "data": {
    "id": "prov-a1b2c3",
    "ip": "172.18.0.5",
    "berat": "hermes-coding-berat",
    "repo": "stranma/EFM",
    "branch": "main",
    "status": "creating"
  }
}
```

**Step 2 -- Resolve default grants from berat security policy.**

Sentinel reads the berat manifest from disk:

```bash
cat /opt/sultanate/berats/hermes-coding-berat/berat.yaml
```

The `security.grants` section defines default grants (see
[HERMES_CODING_BERAT_MVP_PRD.md](HERMES_CODING_BERAT_MVP_PRD.md)):

```yaml
security:
  grants:
    - service: github
      domains:
        - api.github.com
        - github.com
      inject:
        header: Authorization
        value_template: "Bearer {{GITHUB_TOKEN}}"
  whitelist:
    - github.com
    - api.github.com
    - pypi.org
    - files.pythonhosted.org
    - registry.npmjs.org
    - cdn.jsdelivr.net
    - docs.python.org
    - stackoverflow.com
  ports:
    - host: github.com
      port: 22
      protocol: tcp
      reason: "Git SSH"
```

**Step 3 -- Request token values from Sultan via Telegram.**

Sentinel sends a Telegram message to Sultan:

> Province `prov-a1b2c3` (backend-refactor) needs a GitHub token for
> repo `stranma/EFM`. Please provide a fine-grained PAT with read/write
> access to `stranma/EFM` and reply with the token.

Sultan replies with the token value. Sentinel parses the reply.

Per the GitHub Token Strategy in [SULTANATE_MVP.md](SULTANATE_MVP.md),
Sultan pre-creates GitHub PATs (fine-grained, scoped per repo). Sentinel
does not create tokens programmatically.

**Step 4 -- Store token in Infisical:**

```bash
infisical secrets set --projectId sultanate --env prov-a1b2c3 \
  GITHUB_TOKEN="EXAMPLE_GITHUB_TOKEN_HERE"
```

**Step 5 -- Read token back and write grant to Divan:**

For each domain in the grant definition:

```
POST http://127.0.0.1:8600/grants
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
```

Request body for `api.github.com`:
```json
{
  "province_id": "prov-a1b2c3",
  "source_ip": "172.18.0.5",
  "match": {
    "domain": "api.github.com"
  },
  "inject": {
    "header": "Authorization",
    "value": "Bearer EXAMPLE_GITHUB_TOKEN_HERE"
  }
}
```

Repeat for `github.com`:
```json
{
  "province_id": "prov-a1b2c3",
  "source_ip": "172.18.0.5",
  "match": {
    "domain": "github.com"
  },
  "inject": {
    "header": "Authorization",
    "value": "Bearer EXAMPLE_GITHUB_TOKEN_HERE"
  }
}
```

**Step 6 -- Write default whitelist to Divan:**

```
PUT http://127.0.0.1:8600/whitelists/prov-a1b2c3
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
```

```json
{
  "domains": [
    "github.com",
    "api.github.com",
    "pypi.org",
    "files.pythonhosted.org",
    "registry.npmjs.org",
    "cdn.jsdelivr.net",
    "docs.python.org",
    "stackoverflow.com"
  ]
}
```

**Step 7 -- Write port requests to Divan:**

For each non-HTTP port declaration in the berat:

```
POST http://127.0.0.1:8600/port_requests
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
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

Port requests are created in `pending` status. Sultan approval is requested
via Telegram (see §4).

**Step 8 -- Add province to processed set, persist to disk.**

After all grants and whitelist are written, add `prov-a1b2c3` to the
processed provinces set (see §7).

## 3. Grant Revocation Workflow

Triggered when Sentinel's polling loop detects a province with
`status=destroying`.

### Step-by-Step

**Step 1 -- Delete all secrets from Infisical:**

```bash
for secret in $(infisical secrets list --projectId sultanate --env prov-a1b2c3 --plain --silent | awk '{print $1}'); do
  infisical secrets delete "$secret" --projectId sultanate --env prov-a1b2c3
done
```

**Step 2 -- Delete all grants from Divan:**

```
DELETE http://127.0.0.1:8600/grants?province_id=prov-a1b2c3
Authorization: Bearer <DIVAN_KEY_SENTINEL>
```

Response:
```json
{ "data": { "deleted": 2 } }
```

**Step 3 -- Delete whitelist from Divan:**

```
PUT http://127.0.0.1:8600/whitelists/prov-a1b2c3
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
```

```json
{ "domains": [] }
```

Setting to empty array effectively deletes the whitelist.

**Step 4 -- Remove iptables rules for this province.**

Query for the province's container IP from the province record (already read
in polling loop). Remove all iptables rules tagged with the province ID:

```bash
# List rules with province comment tag and delete them
iptables -t nat -S PREROUTING | grep "prov-a1b2c3" | sed 's/-A/-D/' | while read rule; do
  iptables -t nat $rule
done

iptables -S FORWARD | grep "prov-a1b2c3" | sed 's/-A/-D/' | while read rule; do
  iptables $rule
done
```

**Step 5 -- Remove province from processed set, persist to disk.**

**Step 6 -- Notify Sultan via Telegram:**

> Province `prov-a1b2c3` (backend-refactor) revoked. Grants deleted,
> secrets purged, network rules removed.

## 4. Port Request Handling

Sentinel polls Divan for pending port requests and brokers Sultan approval.

### Polling

```
GET http://127.0.0.1:8600/port_requests?status=pending
Authorization: Bearer <DIVAN_KEY_SENTINEL>
```

Polled on the same 10s interval as province polling (see §7).

### Approval Flow

**Step 1 -- For each pending port request, message Sultan via Telegram:**

> Province `prov-a1b2c3` requests TCP access to `github.com:22`.
> Reason: Git SSH.
> Reply APPROVE or DENY.

**Step 2 -- On Sultan's reply:**

If **approved**:

```
PATCH http://127.0.0.1:8600/port_requests/{id}
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
```

```json
{ "status": "approved" }
```

Then open the network route with iptables. Given:
- Province container IP: `172.18.0.5`
- Target host: `github.com` (resolved to IP, e.g., `140.82.121.4`)
- Target port: `22`
- Protocol: `tcp`

```bash
# Resolve target host to IP
TARGET_IP=$(dig +short github.com | head -1)

# Allow FORWARD from container to specific host:port
iptables -A FORWARD \
  -s 172.18.0.5 -d "$TARGET_IP" \
  -p tcp --dport 22 \
  -m comment --comment "prov-a1b2c3" \
  -j ACCEPT

# DNAT rule if needed (route from container's internal network to external)
iptables -t nat -A PREROUTING \
  -s 172.18.0.5 -d "$TARGET_IP" \
  -p tcp --dport 22 \
  -m comment --comment "prov-a1b2c3" \
  -j DNAT --to-destination "$TARGET_IP":22

# Masquerade outbound (so return traffic is routed correctly)
iptables -t nat -A POSTROUTING \
  -s 172.18.0.5 -d "$TARGET_IP" \
  -p tcp --dport 22 \
  -m comment --comment "prov-a1b2c3" \
  -j MASQUERADE
```

The `--comment "prov-a1b2c3"` tag enables batch cleanup during revocation (§3).

If **denied**:

```
PATCH http://127.0.0.1:8600/port_requests/{id}
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
```

```json
{ "status": "denied" }
```

No iptables changes. Sentinel notifies the province agent indirectly -- the
port request status is visible in Divan; Vizier can relay the denial.

## 5. Whitelist and Blacklist Management

Sultan instructs Sentinel via Telegram. Sentinel translates to Divan API
calls.

### Whitelist Operations

**"Whitelist example.com for province prov-a1b2c3":**

```
POST http://127.0.0.1:8600/whitelists/prov-a1b2c3/domains
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
```

```json
{ "domain": "example.com" }
```

**"Remove example.com from province prov-a1b2c3 whitelist":**

```
DELETE http://127.0.0.1:8600/whitelists/prov-a1b2c3/domains/example.com
Authorization: Bearer <DIVAN_KEY_SENTINEL>
```

**"Show whitelist for province prov-a1b2c3":**

```
GET http://127.0.0.1:8600/whitelists/prov-a1b2c3
Authorization: Bearer <DIVAN_KEY_SENTINEL>
```

### Blacklist Operations

**"Add badsite.com to the blacklist":**

```
POST http://127.0.0.1:8600/blacklist/domains
Authorization: Bearer <DIVAN_KEY_SENTINEL>
Content-Type: application/json
```

```json
{ "domain": "badsite.com" }
```

**"Remove badsite.com from the blacklist":**

```
DELETE http://127.0.0.1:8600/blacklist/domains/badsite.com
Authorization: Bearer <DIVAN_KEY_SENTINEL>
```

**"Show the blacklist":**

```
GET http://127.0.0.1:8600/blacklist
Authorization: Bearer <DIVAN_KEY_SENTINEL>
```

All operations are idempotent. Blacklist takes priority over whitelist
(see [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md)).

## 6. Hermes Agent Configuration

### Directory Layout

```
/opt/sultanate/sentinel/          # HERMES_HOME
├── SOUL.md                       # Agent soul
├── config.yaml                   # Hermes configuration
└── state/
    └── processed_provinces.json  # Restart-recovery state (see §7)

/opt/sultanate/sentinel.env       # Infisical + Divan credentials
```

### SOUL.md

```markdown
You are Sentinel, the trusted secret management agent for the Sultanate
system. You run as root on the host with direct network access.

## Your Responsibilities

1. Manage secrets in Infisical for all provinces
2. Provision and revoke grants in Divan based on province lifecycle
3. Handle port access requests -- ask Sultan for approval, then open
   network routes
4. Manage whitelists and blacklist in Divan on Sultan's instruction

## Operating Rules

- Poll Divan every 10 seconds for new provinces and pending port requests
- When you see a new province (status=creating), read its berat security
  policy, ask Sultan for the required token values, store in Infisical,
  write grants and whitelist to Divan
- When you see a destroying province, revoke all secrets, delete all grants
  and whitelist, remove iptables rules
- For port requests, always ask Sultan before approving. Never auto-approve.
- For whitelist/blacklist changes, only act on Sultan's explicit instruction
- Report all actions to Sultan via Telegram

## Security Principles

- Never write secret values into Telegram messages. Confirm receipt with
  masked values (e.g., "Stored token ghp_****1234 for province X")
- Never inject secrets directly into containers. All secrets go through
  Divan grants -> Janissary injection
- If Infisical is unreachable, report to Sultan and retry. Do not fall
  back to storing secrets elsewhere.

## Tools

- Use the terminal tool to run infisical CLI commands
- Use the terminal tool with curl to call Divan API endpoints
- Divan runs at http://127.0.0.1:8600
- Authenticate to Divan with the DIVAN_KEY_SENTINEL bearer token

## Polling Loop

Continuously run this loop:

1. curl GET /provinces -- check for status=creating or status=destroying
2. Process any new or destroying provinces
3. curl GET /port_requests?status=pending -- check for pending requests
4. Process any pending port requests
5. Sleep 10 seconds
6. Repeat

Track processed province IDs in /opt/sultanate/sentinel/state/processed_provinces.json
to avoid re-processing after restarts.
```

### config.yaml

```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"
  provider: openrouter

terminal:
  backend: local
  cwd: "/opt/sultanate/sentinel"
  timeout: 120

tools:
  - terminal

telegram:
  bot_token_env: "SENTINEL_TELEGRAM_BOT_TOKEN"
  allowed_users:
    - "${SULTAN_TELEGRAM_USER_ID}"
```

Only the `terminal` tool is enabled. No `web`, `browser`, `file`, or other
tools. All operations (Infisical CLI, Divan HTTP via curl, iptables) are
executed through the terminal tool.

### Telegram Configuration

Sentinel uses a dedicated Telegram bot (separate from Vizier's bot).
The bot token and Sultan's user ID are provided via environment variables:

- `SENTINEL_TELEGRAM_BOT_TOKEN` -- Bot token from BotFather
- `SULTAN_TELEGRAM_USER_ID` -- Sultan's Telegram user ID (for access control)

These are passed to the Docker container as environment variables (see §9).

## 7. Divan Polling Loop

### Mechanism

Sentinel's SOUL.md instructs it to run a continuous polling loop using
terminal tool commands (curl + sleep). The loop is the agent's primary
behavior -- it polls, processes, and repeats.

### Poll Interval

10 seconds between iterations.

### Loop Pseudocode

```
load processed_set from /opt/sultanate/sentinel/state/processed_provinces.json

loop:
  # 1. Poll provinces
  provinces = GET /provinces
  
  for each province in provinces:
    if province.status == "creating" AND province.id NOT IN processed_set:
      run grant provisioning workflow (§2)
      add province.id to processed_set with state "provisioned"
      save processed_set to disk
    
    if province.status == "destroying" AND province.id IN processed_set:
      run grant revocation workflow (§3)
      remove province.id from processed_set
      save processed_set to disk
  
  # 2. Poll port requests
  pending_ports = GET /port_requests?status=pending
  
  for each request in pending_ports:
    if request.id NOT IN pending_port_request_set:
      add request.id to pending_port_request_set
      ask Sultan for approval via Telegram (§4)
  
  # 3. Sleep
  sleep 10
```

### Restart Recovery

Processed provinces are persisted to:

```
/opt/sultanate/sentinel/state/processed_provinces.json
```

Format:
```json
{
  "prov-a1b2c3": { "state": "provisioned", "provisioned_at": "2026-04-12T10:31:00Z" },
  "prov-d4e5f6": { "state": "provisioned", "provisioned_at": "2026-04-12T11:05:00Z" }
}
```

On startup, Sentinel loads this file. If the file is missing or corrupt,
Sentinel starts with an empty set and re-polls. This may cause Sultan to
receive duplicate token requests for already-provisioned provinces. Sultan
can reply "already provisioned" or provide the token again (idempotent
storage in Infisical).

Port request tracking is ephemeral (not persisted). On restart, Sentinel
re-reads pending port requests from Divan and re-asks Sultan. This is
acceptable because Sultan approval is idempotent.

## 8. Error Handling

### Infisical Unreachable

- **Detection:** Non-zero exit code from `infisical` CLI commands.
- **Action:** Report to Sultan via Telegram: "Infisical is unreachable.
  Cannot provision/revoke secrets. Retrying."
- **Retry:** 3 attempts with 10s backoff (10s, 20s, 30s).
- **Escalation:** After 3 failures, message Sultan: "Infisical still down
  after 3 retries. Province prov-X provisioning deferred. Will retry on
  next poll cycle."
- **Province left in `creating` state.** Sentinel does not add it to the
  processed set, so it will be retried on the next poll cycle.

### Divan Unreachable

- **Detection:** curl returns non-200 or connection refused.
- **Action:** Report to Sultan via Telegram: "Divan is unreachable.
  Polling suspended."
- **Retry:** Sentinel's poll loop continues. Each 10s iteration retries
  the Divan connection. No exponential backoff -- the poll loop itself
  is the retry mechanism.
- **Impact:** No provisioning, revocation, or port request processing
  until Divan recovers. Existing iptables rules and Infisical secrets
  remain intact.

### Sultan Doesn't Respond to Token Request

- **Timeout:** No hard timeout. Sentinel's poll loop continues processing
  other provinces and port requests.
- **Province remains in `creating` state** in Divan. Sentinel tracks it
  as "awaiting Sultan response" in its internal state (not persisted).
- **Reminder:** Every 5 poll cycles (50s) while awaiting response, Sentinel
  sends a reminder: "Still waiting for GitHub token for province prov-X."
- **No auto-provisioning.** Sentinel never creates tokens or grants without
  Sultan providing the actual secret value.
- **Cancellation:** Sultan can reply "cancel" to abandon the province
  setup. Sentinel removes it from the pending queue.

### Sultan Doesn't Respond to Port Request

- Same as token request: no hard timeout, periodic reminders, port request
  stays `pending` in Divan.
- Province can still function for HTTP traffic (via Janissary). Only the
  specific non-HTTP port remains blocked.

### Partial Provisioning Failure

If Infisical write succeeds but Divan grant write fails:
- Sentinel retries the Divan write (3 attempts, 10s backoff).
- On persistent failure, secrets remain in Infisical but grants are not
  in Divan. Janissary will not inject credentials.
- Sentinel reports to Sultan: "Grant write to Divan failed for province
  prov-X. Secrets stored but not activated."
- On next poll cycle, if province is still `creating` and not in processed
  set, Sentinel retries the full workflow (reads existing Infisical secret,
  re-attempts Divan write).

### Partial Revocation Failure

If Infisical delete succeeds but Divan grant delete fails:
- Sentinel retries the Divan delete (3 attempts, 10s backoff).
- On persistent failure, stale grants may remain in Divan. Janissary
  would inject expired/deleted tokens (benign -- requests would fail at
  the upstream service).
- Sentinel reports to Sultan and retries on next poll cycle.

## 9. Deployment

### Docker Run Command

```bash
docker run -d \
  --name sentinel \
  --network host \
  --restart unless-stopped \
  -v /opt/sultanate/sentinel:/opt/data:rw \
  -v /opt/sultanate/berats:/opt/sultanate/berats:ro \
  -v /opt/sultanate/sentinel.env:/opt/sultanate/sentinel.env:ro \
  -e SENTINEL_TELEGRAM_BOT_TOKEN="<bot-token>" \
  -e SULTAN_TELEGRAM_USER_ID="<sultan-user-id>" \
  -e DIVAN_KEY_SENTINEL="<divan-api-key>" \
  -e INFISICAL_API_URL="http://127.0.0.1:8070/api" \
  --cap-add NET_ADMIN \
  nousresearch/hermes-agent
```

### docker-compose.yml Snippet

```yaml
services:
  sentinel:
    image: nousresearch/hermes-agent
    container_name: sentinel
    network_mode: host
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    volumes:
      - /opt/sultanate/sentinel:/opt/data:rw
      - /opt/sultanate/berats:/opt/sultanate/berats:ro
      - /opt/sultanate/sentinel.env:/opt/sultanate/sentinel.env:ro
    environment:
      - SENTINEL_TELEGRAM_BOT_TOKEN=${SENTINEL_TELEGRAM_BOT_TOKEN}
      - SULTAN_TELEGRAM_USER_ID=${SULTAN_TELEGRAM_USER_ID}
      - DIVAN_KEY_SENTINEL=${DIVAN_KEY_SENTINEL}
      - INFISICAL_API_URL=http://127.0.0.1:8070/api
    depends_on:
      divan:
        condition: service_healthy
```

### Key Configuration Details

| Setting | Value | Reason |
|---------|-------|--------|
| `network_mode: host` | Direct host networking | Needs Telegram API, Infisical (localhost), Divan (localhost), iptables |
| `cap_add: NET_ADMIN` | Linux capability | Required for iptables rule management |
| `HERMES_HOME` | `/opt/data` (mapped to `/opt/sultanate/sentinel`) | Hermes convention: SOUL.md and config.yaml live here |
| Berats volume | Read-only mount | Sentinel reads berat security policies but never writes them |
| `restart: unless-stopped` | Auto-restart | Sentinel must survive crashes; state file enables recovery (§7) |

### Environment Variables

| Variable | Source | Description |
|----------|--------|-------------|
| `SENTINEL_TELEGRAM_BOT_TOKEN` | BotFather | Dedicated Sentinel bot token |
| `SULTAN_TELEGRAM_USER_ID` | Telegram | Sultan's user ID for access control |
| `DIVAN_KEY_SENTINEL` | `/opt/sultanate/divan.env` | Pre-shared API key for Divan |
| `INFISICAL_API_URL` | Static | Infisical API endpoint (localhost) |
| `INFISICAL_CLIENT_ID` | `/opt/sultanate/sentinel.env` | Machine Identity client ID |
| `INFISICAL_CLIENT_SECRET` | `/opt/sultanate/sentinel.env` | Machine Identity client secret |

### Startup Order

Per [SULTANATE_MVP.md](SULTANATE_MVP.md):

```
1. Divan       (shared state)
2. Janissary   (proxy)
3. Sentinel    (this service)
4. Vizier      (management)
5. Provinces   (on demand)
```

Sentinel depends on Divan being healthy. It does not depend on Janissary
(direct host networking). Sentinel starts before Vizier -- grants must be
writable before Vizier creates provinces.

### Pre-deployment Checklist

1. Infisical running on host at `127.0.0.1:8070`
2. Infisical project `sultanate` created with Machine Identity configured
3. `/opt/sultanate/sentinel.env` created with Infisical credentials
4. `/opt/sultanate/divan.env` created with `DIVAN_KEY_SENTINEL`
5. `/opt/sultanate/sentinel/SOUL.md` written (§6)
6. `/opt/sultanate/sentinel/config.yaml` written (§6)
7. `/opt/sultanate/sentinel/state/` directory created
8. `/opt/sultanate/berats/hermes-coding-berat/berat.yaml` present
9. Sentinel Telegram bot created via BotFather
10. Divan service healthy (`GET /health` returns 200)
