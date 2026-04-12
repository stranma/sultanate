# Janissary Technical Specification

> Egress proxy for the Sultanate platform. For requirements see
> [JANISSARY_MVP_PRD.md](JANISSARY_MVP_PRD.md). For shared state contract see
> [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md). For architecture context see
> [SULTANATE_MVP.md](SULTANATE_MVP.md).

---

## 1. Sandcat Integration Approach

[Sandcat](https://github.com/VirtusLab/sandcat) (VirtusLab) is a
mitmproxy-based dev container sandbox that provides:

- **Transparent proxy via WireGuard** -- all container traffic routed through
  mitmproxy without per-tool proxy configuration
- **Python mitmproxy addon** (`mitmproxy_addon.py`) with `request()` and
  `dns_request()` hooks for network rule evaluation (allow/deny, first-match-wins
  via `fnmatch` glob patterns)
- **Secret substitution** -- placeholder strings in env vars
  (`SANDCAT_PLACEHOLDER_<NAME>`) replaced with real values at proxy level,
  scoped to allowed destination hosts
- **Docker Compose setup** -- `wg-client` (WireGuard + iptables kill-switch),
  `mitmproxy` container, shared volume for CA cert and env

### What Janissary reuses from Sandcat

Janissary adopts Sandcat's **mitmproxy addon architecture pattern**:

| Sandcat pattern | Janissary equivalent |
|-----------------|---------------------|
| `SandcatAddon` class with `request()` hook | `JanissaryAddon` class with `request()` and `http_connect()` hooks |
| `_is_request_allowed(method, host)` | `_evaluate_traffic_rules(source_ip, method, host)` |
| `_substitute_secrets()` placeholder replacement | `_inject_credentials()` header injection from Divan grants |
| File-based settings (`settings.json`) | Divan polling (`GET /janissary/state`) |
| `fnmatch` glob domain matching | Exact domain string equality |
| `dns_request()` hook for DNS filtering | Not used (provinces use HTTP_PROXY, not transparent mode) |
| `load()` reads settings once at startup | `running()` starts background polling thread |

### What Janissary does NOT reuse

- **WireGuard transparent mode** -- Janissary runs mitmproxy in **regular
  forward proxy mode** (`--mode regular`). Provinces set `HTTP_PROXY` /
  `HTTPS_PROXY` env vars. No WireGuard, no `wg-client` container, no
  `NET_ADMIN` capability on province containers.
- **Sandcat CLI / settings layering** -- Janissary reads state from Divan,
  not from JSON settings files.
- **Placeholder-based secret substitution** -- Janissary uses direct HTTP
  header injection (province never sees a placeholder or real value).
- **Per-project / per-user settings** -- All state comes from Divan, written
  by Sentinel and Vizier.

### Relationship to Sandcat codebase

Janissary is **not a fork and not a wrapper**. It is a standalone mitmproxy
addon (single Python file + a FastAPI appeal API server) that follows the
same architectural pattern as Sandcat's addon. The Sandcat repo is
referenced for design guidance, not imported as a dependency.

```
janissary/
├── janissary_addon.py    # mitmproxy addon (traffic rules, credential injection)
├── janissary_api.py      # FastAPI appeal/access-request HTTP API
├── janissary_state.py    # Divan poller + in-memory cache
├── janissary_config.py   # Config loading
├── janissary_main.py     # Process entry point (starts mitmproxy + API server)
├── Dockerfile
└── requirements.txt      # mitmproxy, fastapi, uvicorn, httpx
```

---

## 2. CA Certificate Lifecycle

### Generation

A single Sultanate-wide CA certificate and private key are generated **once
at deploy time** by the deploy/bootstrap script (not by Janissary itself):

```bash
#!/bin/bash
# /opt/sultanate/scripts/generate-ca.sh
CA_DIR="/opt/sultanate/certs"
mkdir -p "$CA_DIR"

openssl req -x509 -new -nodes \
  -keyout "$CA_DIR/sultanate-ca.key" \
  -out "$CA_DIR/sultanate-ca.pem" \
  -days 3650 \
  -subj "/CN=Sultanate CA/O=Sultanate"

chmod 600 "$CA_DIR/sultanate-ca.key"
chmod 644 "$CA_DIR/sultanate-ca.pem"
```

### Storage on host

```
/opt/sultanate/certs/
├── sultanate-ca.pem       # CA certificate (world-readable)
└── sultanate-ca.key       # CA private key (root + janissary container only)
```

The CA key must be readable by the mitmproxy process inside the Janissary
container. It is bind-mounted read-only into Janissary's container.

### Distribution to provinces

Vizier mounts the CA cert (not the key) into every province container at
creation time:

```yaml
# In province container creation (Vizier sets this)
volumes:
  - /opt/sultanate/certs/sultanate-ca.pem:/usr/local/share/ca-certificates/sultanate-ca.crt:ro
```

### Trust establishment in containers

The firman entrypoint (`app-init.sh`) runs at province container startup:

```bash
# Install CA into system trust store (Debian/Ubuntu)
update-ca-certificates

# Node.js: set NODE_EXTRA_CA_CERTS (Node bundles its own CA store)
export NODE_EXTRA_CA_CERTS="/usr/local/share/ca-certificates/sultanate-ca.crt"

# Python: uses system store -- works out of the box after update-ca-certificates
# Java: import into cacerts if JRE present
if command -v keytool &>/dev/null; then
  keytool -importcert -trustcacerts -noprompt \
    -alias sultanate-ca \
    -file /usr/local/share/ca-certificates/sultanate-ca.crt \
    -keystore "$JAVA_HOME/lib/security/cacerts" \
    -storepass changeit 2>/dev/null || true
fi
```

### mitmproxy CA configuration

mitmproxy is started with `--set confdir=/opt/mitmproxy` where the CA files
are pre-placed. mitmproxy expects:

```
/opt/mitmproxy/
├── mitmproxy-ca.pem           # symlink -> /certs/sultanate-ca.pem
└── mitmproxy-ca.p12           # generated from the CA cert+key at container start
```

The Janissary Dockerfile entrypoint converts the PEM cert+key into the
formats mitmproxy expects:

```bash
# janissary-entrypoint.sh
CONFDIR="/opt/mitmproxy"
mkdir -p "$CONFDIR"
cp /certs/sultanate-ca.pem "$CONFDIR/mitmproxy-ca-cert.pem"
cp /certs/sultanate-ca.key "$CONFDIR/mitmproxy-ca.pem"
cat /certs/sultanate-ca.key /certs/sultanate-ca.pem > "$CONFDIR/mitmproxy-ca.pem"

openssl pkcs12 -export -nokeys \
  -in /certs/sultanate-ca.pem \
  -out "$CONFDIR/mitmproxy-ca-cert.p12" \
  -passout pass:""
```

---

## 3. Proxy Configuration

### Listen address and port

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| mitmproxy | `8080` | HTTP proxy (CONNECT for HTTPS) | Forward proxy for all province/Vizier traffic |
| Appeal API | `8081` | HTTP | Appeal and access-request endpoints |

mitmproxy listens on `0.0.0.0:8080` inside the Janissary container.
The appeal API (FastAPI + Uvicorn) listens on `0.0.0.0:8081`.

### Docker network setup

```
sultanate-internal (bridge, internal: true)
├── janissary       172.18.0.2
├── vizier          172.18.0.3
├── province-1      172.18.0.5   (dynamic, registered in Divan)
├── province-2      172.18.0.6
└── ...

sultanate-external (bridge, internal: false)
├── janissary       172.19.0.2   (dual-homed: internal + external)
```

Janissary is the **only container on both networks**. Provinces and Vizier
are on `sultanate-internal` only (`internal: true` -- no default route to the
internet). Janissary's second interface on `sultanate-external` provides
the actual internet egress path.

### Province proxy configuration

Vizier sets these environment variables on every province container:

```bash
HTTP_PROXY=http://172.18.0.2:8080
HTTPS_PROXY=http://172.18.0.2:8080
NO_PROXY=172.18.0.2
```

`NO_PROXY=172.18.0.2` ensures that requests to Janissary's appeal API
(`http://172.18.0.2:8081/api/...`) go directly, not through the proxy.

Vizier's own environment has the same variables.

### Janissary configuration file

`/opt/sultanate/janissary/config.yaml`:

```yaml
# Janissary configuration
proxy:
  listen_host: "0.0.0.0"
  listen_port: 8080

api:
  listen_host: "0.0.0.0"
  listen_port: 8081

divan:
  url: "http://127.0.0.1:8600"
  api_key_env: "DIVAN_KEY_JANISSARY"     # reads from environment
  poll_interval_seconds: 5
  health_check_interval_seconds: 2
  health_check_timeout_seconds: 60

ca:
  cert_path: "/certs/sultanate-ca.pem"
  key_path: "/certs/sultanate-ca.key"
  confdir: "/opt/mitmproxy"

logging:
  level: "INFO"                           # DEBUG, INFO, WARNING, ERROR
  audit_file: "/var/log/janissary/audit.jsonl"
```

### mitmproxy launch command

```bash
mitmdump \
  --mode regular \
  --listen-host 0.0.0.0 \
  --listen-port 8080 \
  --set confdir=/opt/mitmproxy \
  --set connection_strategy=lazy \
  -s /opt/janissary/janissary_addon.py
```

`mitmdump` (not `mitmweb`) -- no web UI needed in production. The
`connection_strategy=lazy` setting delays upstream connections until the
addon has decided whether to allow the request (avoids unnecessary
connections for blocked requests).

---

## 4. Traffic Rule Implementation

### Rule evaluation order

Evaluated per-request in the `request()` mitmproxy hook:

```python
def _evaluate_traffic_rules(self, source_ip: str, method: str, host: str) -> str:
    """Returns: 'allow', 'block_blacklist', 'block_write', 'block_no_state'"""

    # Rule 0: No cached state (fresh start, never polled Divan) -> fail closed
    if not self.state.has_loaded:
        return "block_no_state"

    # Rule 1: Blacklist (global, all methods)
    if host in self.state.blacklist:
        return "block_blacklist"

    # Rule 2: Whitelist (per-source, all methods)
    source_whitelist = self.state.get_whitelist(source_ip)
    if host in source_whitelist:
        return "allow"

    # Rule 3: Read-only pass (GET/HEAD to non-whitelisted domain)
    if method.upper() in ("GET", "HEAD"):
        return "allow"

    # Rule 4: Write block (POST/PUT/PATCH/DELETE to non-whitelisted domain)
    # Check one-time approved appeals first
    if self.state.has_approved_appeal(source_ip, method, host):
        return "allow"

    return "block_write"
```

### Domain matching semantics

**Exact string equality on the domain label.** No glob patterns, no
wildcard subdomain matching, no regex.

```
Whitelist contains: "github.com"

github.com       → match ✓
api.github.com   → no match ✗  (must be listed separately)
www.github.com   → no match ✗
GITHUB.COM       → match ✓     (case-insensitive comparison)
github.com.      → match ✓     (trailing dot stripped before comparison)
```

Implementation:

```python
def _normalize_domain(self, domain: str) -> str:
    return domain.lower().rstrip(".")
```

All domain comparisons use normalized forms. Divan stores domains in
lowercase without trailing dots.

### HTTP vs HTTPS request handling

#### HTTP requests (no TLS)

The `request()` hook fires with full request details (method, host, path,
headers, body). All 4 rules are evaluated. Credential injection applies.

#### HTTPS requests (TLS via CONNECT)

HTTPS flows through two hooks:

1. **`http_connect()`** -- fires when the client sends `CONNECT host:443`.
   Janissary decides here whether to MITM or pass-through:

```python
def http_connect(self, flow: http.HTTPFlow):
    host = self._normalize_domain(flow.request.pretty_host)
    source_ip = flow.client_conn.peername[0]

    # Blacklisted -> block immediately (respond 403 to CONNECT)
    if host in self.state.blacklist:
        flow.response = http.Response.make(403, ...)
        return

    # Grant exists for this source+domain -> MITM (need to inject headers)
    if self.state.has_grant(source_ip, host):
        # Do nothing: mitmproxy will MITM by default
        return

    # No grant -> pass-through (TLS tunnel, no inspection)
    # Whitelisted domains: safe to pass through (all methods allowed)
    # Non-whitelisted domains: pass through (can't inspect method)
    flow.metadata["passthrough"] = True
```

2. **`request()`** -- fires only for MITM'd connections. Full rule evaluation
   and credential injection apply.

For pass-through HTTPS (no MITM):
- Blacklisted domains: blocked at `http_connect()`
- Whitelisted domains: passed through (all methods allowed per rule 2)
- Non-whitelisted domains without grants: passed through -- **method-based
  rules (3, 4) cannot be enforced** since the HTTP method is inside the
  encrypted tunnel

> **Security note:** Write-block (rule 4) is not enforced for HTTPS to
> non-whitelisted domains without grants. This is a deliberate trade-off:
> MITM'ing all HTTPS adds CA trust complexity and performance overhead.
> Agents writing to non-whitelisted HTTPS endpoints without injected
> credentials have limited ability to authenticate, reducing exfiltration
> risk. Audit logs record all CONNECT requests for review.

### mitmproxy passthrough implementation

To pass-through (not MITM) a specific connection, use mitmproxy's
`ignore_hosts` or set the flow to non-intercepted:

```python
def tls_clienthello(self, flow: tls.ClientHelloData):
    """Skip MITM for connections marked as passthrough."""
    if flow.context.metadata.get("passthrough"):
        flow.ignore_connection = True
```

### Audit logging

Every request decision is logged to the audit file as JSONL:

```json
{
  "ts": "2026-04-12T11:00:00.123Z",
  "source_ip": "172.18.0.5",
  "province_id": "prov-a1b2c3",
  "method": "POST",
  "host": "api.github.com",
  "path": "/repos/stranma/EFM/pulls",
  "decision": "allow",
  "rule": "whitelist",
  "mitm": true,
  "credential_injected": true
}
```

---

## 5. Divan Polling and Caching

### Polling mechanism

A background thread polls Divan's bulk state endpoint:

```python
class DivanPoller:
    def __init__(self, config: JanissaryConfig):
        self.divan_url = config.divan.url        # http://127.0.0.1:8600
        self.api_key = os.environ[config.divan.api_key_env]
        self.poll_interval = config.divan.poll_interval_seconds  # 5
        self.has_loaded = False                   # True after first successful poll
        self._cache = JanissaryStateCache()
        self._lock = threading.RLock()

    def poll_loop(self):
        while self._running:
            try:
                resp = httpx.get(
                    f"{self.divan_url}/janissary/state",
                    headers={"Authorization": f"Bearer {self.api_key}"},
                    timeout=10.0,
                )
                resp.raise_for_status()
                data = resp.json()["data"]
                with self._lock:
                    self._cache = JanissaryStateCache.from_divan_response(data)
                    self.has_loaded = True
            except Exception as e:
                logger.warning(f"Divan poll failed: {e}")
                # Keep using cached state; has_loaded remains False if never loaded
            time.sleep(self.poll_interval)
```

### Cache data structure

```python
class JanissaryStateCache:
    # IP -> province_id mapping
    ip_to_province: dict[str, str]

    # Global blacklist: set of normalized domains
    blacklist: set[str]

    # Per-source whitelists: source_id -> set of normalized domains
    # Keys are province IDs from Divan; lookup by IP uses ip_to_province mapping
    whitelists: dict[str, set[str]]

    # Grants indexed for fast lookup: (source_ip, domain) -> Grant
    grants: dict[tuple[str, str], Grant]

    # Approved one-time appeals: (source_ip, normalized_url, method) -> resolved_at
    approved_appeals: dict[tuple[str, str, str], str]

    @staticmethod
    def from_divan_response(data: dict) -> "JanissaryStateCache":
        cache = JanissaryStateCache()

        # Build IP -> province_id mapping
        cache.ip_to_province = {
            p["ip"]: p["id"]
            for p in data["provinces"]
            if p.get("ip") and p.get("status") == "running"
        }

        # Blacklist
        cache.blacklist = {d.lower().rstrip(".") for d in data["blacklist"]}

        # Whitelists: key by province_id, normalize domains
        cache.whitelists = {
            source_id: {d.lower().rstrip(".") for d in domains}
            for source_id, domains in data["whitelists"].items()
        }

        # Grants: index by (source_ip, domain)
        cache.grants = {}
        for g in data["grants"]:
            key = (g["source_ip"], g["match"]["domain"].lower().rstrip("."))
            cache.grants[key] = Grant(
                header=g["inject"]["header"],
                value=g["inject"]["value"],
            )

        # Approved appeals (one-time, within 5-min window)
        cache.approved_appeals = {}
        for a in data["approved_appeals"]:
            url_host = _extract_host_from_url(a["url"])
            key = (a["source_ip"], a["url"], a["method"].upper())
            cache.approved_appeals[key] = a["resolved_at"]

        return cache
```

### Lookup methods

```python
def get_whitelist(self, source_ip: str) -> set[str]:
    """Get whitelist domains for a source IP."""
    province_id = self.ip_to_province.get(source_ip)
    if not province_id:
        return set()
    return self.whitelists.get(province_id, set())

def has_grant(self, source_ip: str, domain: str) -> bool:
    return (source_ip, domain) in self.grants

def get_grant(self, source_ip: str, domain: str) -> Grant | None:
    return self.grants.get((source_ip, domain))

def has_approved_appeal(self, source_ip: str, method: str, host: str) -> bool:
    """Check if there's a one-time approved appeal matching this request.
    Divan's bulk state endpoint already filters to 5-minute window."""
    for (a_ip, a_url, a_method), _ in self.approved_appeals.items():
        if a_ip == source_ip and a_method == method.upper():
            a_host = _extract_host_from_url(a_url)
            if a_host == host:
                return True
    return False

def get_province_id(self, source_ip: str) -> str | None:
    return self.ip_to_province.get(source_ip)
```

### Fail-closed behavior

| Scenario | Behavior |
|----------|----------|
| Fresh start, never polled Divan | **Block all traffic.** `has_loaded = False` → every request returns 403 with "Janissary initializing, waiting for state" |
| Divan unreachable after successful poll | **Use cached state.** Last-known rules remain in effect. Log warnings. |
| Divan returns error (500, etc.) | Same as unreachable -- keep cached state, log warning |
| Province IP not in cache | Traffic from unknown IPs is blocked. Only registered, running provinces are served. |

### Polling interval

Default: **5 seconds** (configurable via `config.yaml`). This means:
- Whitelist additions take up to 5s to take effect
- One-time appeal approvals are available within 5s of Sultan's decision
- Blacklist changes propagate within 5s

---

## 6. Credential Injection

### How it works

When mitmproxy MITM's an HTTPS connection (because a grant exists for
source_ip + domain), the `request()` hook has full access to the decrypted
HTTP request. Janissary injects the credential header before forwarding:

```python
def _inject_credentials(self, flow: http.HTTPFlow, source_ip: str):
    host = self._normalize_domain(flow.request.pretty_host)
    grant = self.state.get_grant(source_ip, host)
    if grant is None:
        return

    # Add or replace the header specified in the grant
    flow.request.headers[grant.header] = grant.value
```

### MITM decision flow for HTTPS

```
Client CONNECT api.github.com:443
  │
  ├── Blacklisted? → 403 (blocked)
  │
  ├── Grant exists for (source_ip, api.github.com)?
  │     YES → MITM the connection
  │           → mitmproxy terminates TLS with Sultanate CA cert
  │           → Client re-establishes TLS with mitmproxy (trusts CA)
  │           → mitmproxy opens new TLS connection to api.github.com
  │           → request() hook fires:
  │               → Evaluate traffic rules (blacklist/whitelist/read-only/write-block)
  │               → If allowed: inject Authorization header from grant
  │               → Forward to upstream
  │               → Return response to client
  │
  └── No grant → Pass-through
        → mitmproxy tunnels raw TCP/TLS (no inspection)
        → Blacklist checked at CONNECT level
        → Whitelisted domains: pass through
        → Non-whitelisted domains: pass through (method unknown)
```

### Grant matching

Grant lookup uses exact match on `(source_ip, normalized_domain)`:

```
Grant: source_ip=172.18.0.5, domain=api.github.com
Request from 172.18.0.5 to api.github.com  → injected ✓
Request from 172.18.0.5 to github.com      → NOT injected (different domain)
Request from 172.18.0.6 to api.github.com  → NOT injected (different source)
```

### What the province sees

The province **never** sees the injected credential value. From the
province's perspective:

1. Province sends `POST https://api.github.com/repos/...` (no auth header)
2. Janissary intercepts (MITM), adds `Authorization: Bearer ghp_xxxx`
3. Upstream GitHub receives the authenticated request
4. Response flows back through Janissary to the province (unmodified)

If the province includes its own `Authorization` header, the grant's
`inject` **replaces** it (set, not append).

---

## 7. Appeal Mechanics

### When appeals happen

A write request (POST/PUT/PATCH/DELETE) to a non-whitelisted domain is
blocked by rule 4. The 403 response body tells the agent how to appeal.

### Appeal API server

Janissary runs a **separate FastAPI HTTP server** on port `8081` alongside
mitmproxy (port `8080`). Both run in the same container, started by
`janissary_main.py`.

The appeal API is **not** an MCP server package. It is a plain HTTP API.
MCP tools in provinces call it via standard HTTP requests.

### Endpoints

#### `POST /api/appeal`

Request an exception for a blocked write request.

**Request:**
```json
{
  "url": "https://api.github.com/repos/stranma/EFM/pulls",
  "method": "POST",
  "justification": "Need to submit the PR review comment for task #42"
}
```

Janissary identifies the caller by source IP (from the TCP connection).

**Processing:**
1. Extract `source_ip` from the request's TCP connection
2. Look up `province_id` from the state cache
3. Write to Divan: `POST /appeals` with `source_ip`, `province_id`, `url`,
   `method`, `justification`
4. Return response

**Response (201):**
```json
{
  "status": "pending",
  "appeal_id": "appeal-m1n2o3",
  "message": "Appeal submitted. Waiting for Sultan's decision. Retry your request within 5 minutes of approval."
}
```

**Response (400):** missing or invalid fields.
**Response (503):** Divan unreachable.

#### `POST /api/request_access`

Request new credentials or permanent whitelist addition.

**Request:**
```json
{
  "service": "api.github.com",
  "scope": "repo:write for stranma/EFM",
  "justification": "Need push access to submit PRs"
}
```

**Processing:**
1. Extract `source_ip`, look up `province_id`
2. Write to Divan: `POST /appeals` with `url` set to
   `access-request://{service}`, `method` set to `ACCESS_REQUEST`,
   and the `justification`
3. Vizier picks this up and routes to Sultan via Telegram

**Response (201):**
```json
{
  "status": "pending",
  "message": "Access request submitted. Sultan will be notified via Vizier."
}
```

### Province access to appeal API

Province containers have `NO_PROXY=172.18.0.2` (Janissary's IP), so
requests to `http://172.18.0.2:8081/api/appeal` bypass the proxy and go
directly to the appeal API server.

The MCP tool in provinces (provided by the firman) calls the API:

```python
# In province MCP tool implementation
import httpx

JANISSARY_API = "http://172.18.0.2:8081"

def appeal_request(url: str, method: str, justification: str) -> dict:
    resp = httpx.post(f"{JANISSARY_API}/api/appeal", json={
        "url": url,
        "method": method,
        "justification": justification,
    })
    return resp.json()

def request_access(service: str, scope: str, justification: str) -> dict:
    resp = httpx.post(f"{JANISSARY_API}/api/request_access", json={
        "service": service,
        "scope": scope,
        "justification": justification,
    })
    return resp.json()
```

### One-time approval retry flow

```
1. Agent POSTs to api.example.com → blocked (rule 4) → gets 403
2. Agent calls appeal_request("https://api.example.com/data", "POST", "...")
3. Janissary writes appeal to Divan (status: pending)
4. Vizier polls Divan, relays to Sultan via Telegram
5. Sultan approves (one-time) via Vizier
6. Vizier writes to Divan: PATCH /appeals/{id} status=approved, decision=one-time
7. Janissary's next poll picks up the approval in approved_appeals
8. Agent retries POST to api.example.com
9. Janissary checks _evaluate_traffic_rules():
   - Not blacklisted ✓
   - Not whitelisted → check read-only → POST is not GET/HEAD → check appeals
   - has_approved_appeal(source_ip, "POST", "api.example.com") → True
   - Result: allow
10. Request goes through (one time only)
```

The **5-minute window** is enforced server-side by Divan: the
`/janissary/state` endpoint only includes `approved_appeals` with
`resolved_at` within the last 5 minutes. After 5 minutes, the approval
expires and the agent must appeal again.

---

## 8. 403 Response Body

### Write-block response (rule 4)

When a write is blocked to a non-whitelisted domain:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "write_blocked",
  "message": "Write request blocked: POST to api.example.com is not on your whitelist.",
  "details": {
    "method": "POST",
    "host": "api.example.com",
    "path": "/data",
    "source_ip": "172.18.0.5",
    "province_id": "prov-a1b2c3",
    "rule": "write_block"
  },
  "appeal": {
    "url": "http://172.18.0.2:8081/api/appeal",
    "method": "POST",
    "content_type": "application/json",
    "body_template": {
      "url": "https://api.example.com/data",
      "method": "POST",
      "justification": "<explain why you need this>"
    },
    "instructions": "Submit an appeal by POSTing to the appeal URL above. Include a justification explaining why this write is needed. After approval, retry your original request within 5 minutes."
  }
}
```

### Blacklist-block response (rule 1)

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "blacklisted",
  "message": "Domain pastebin.com is on the global blacklist. All traffic blocked.",
  "details": {
    "host": "pastebin.com",
    "source_ip": "172.18.0.5",
    "rule": "blacklist"
  }
}
```

Blacklisted domains **cannot** be appealed.

### No-state response (fresh start)

```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "error": "initializing",
  "message": "Janissary is starting up and has not yet loaded state from Divan. Please retry shortly."
}
```

---

## 9. Docker Deployment

### docker-compose.yml snippet

```yaml
services:
  divan:
    build:
      context: ./divan
    volumes:
      - /opt/sultanate/divan.db:/opt/sultanate/divan.db
      - /opt/sultanate/divan.env:/opt/sultanate/divan.env:ro
    network_mode: host          # localhost only, port 8600
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://127.0.0.1:8600/health"]
      interval: 2s
      timeout: 2s
      retries: 15

  janissary:
    build:
      context: ./janissary
    depends_on:
      divan:
        condition: service_healthy
    volumes:
      - /opt/sultanate/certs/sultanate-ca.pem:/certs/sultanate-ca.pem:ro
      - /opt/sultanate/certs/sultanate-ca.key:/certs/sultanate-ca.key:ro
      - /opt/sultanate/janissary/config.yaml:/opt/janissary/config.yaml:ro
      - /opt/sultanate/divan.env:/opt/sultanate/divan.env:ro
      - janissary-logs:/var/log/janissary
    env_file:
      - /opt/sultanate/divan.env      # provides DIVAN_KEY_JANISSARY
    networks:
      sultanate-internal:
        ipv4_address: 172.18.0.2
      sultanate-external: {}
    ports: []                          # no host-exposed ports
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python3", "-c",
        "import httpx; r=httpx.get('http://127.0.0.1:8081/health'); r.raise_for_status()"]
      interval: 5s
      timeout: 3s
      retries: 10

networks:
  sultanate-internal:
    driver: bridge
    internal: true                     # no external route
    ipam:
      config:
        - subnet: 172.18.0.0/16
          gateway: 172.18.0.1

  sultanate-external:
    driver: bridge
    internal: false                    # has external route

volumes:
  janissary-logs:
```

### Janissary Dockerfile

```dockerfile
FROM python:3.12-slim

RUN pip install --no-cache-dir \
    mitmproxy==11.* \
    fastapi==0.115.* \
    uvicorn[standard]==0.34.* \
    httpx==0.28.* \
    pyyaml==6.*

COPY . /opt/janissary/
WORKDIR /opt/janissary

RUN mkdir -p /opt/mitmproxy /var/log/janissary

EXPOSE 8080 8081

ENTRYPOINT ["/opt/janissary/janissary-entrypoint.sh"]
```

### janissary-entrypoint.sh

```bash
#!/bin/bash
set -euo pipefail

# Prepare mitmproxy confdir with Sultanate CA
CONFDIR="/opt/mitmproxy"
mkdir -p "$CONFDIR"
cat /certs/sultanate-ca.key /certs/sultanate-ca.pem > "$CONFDIR/mitmproxy-ca.pem"
cp /certs/sultanate-ca.pem "$CONFDIR/mitmproxy-ca-cert.pem"

# Wait for Divan to be healthy
echo "Waiting for Divan..."
until curl -sf http://127.0.0.1:8600/health > /dev/null 2>&1; do
  sleep 1
done
echo "Divan is ready."

# Start appeal API in background
python3 /opt/janissary/janissary_api.py &
API_PID=$!

# Start mitmproxy (foreground)
exec mitmdump \
  --mode regular \
  --listen-host 0.0.0.0 \
  --listen-port 8080 \
  --set confdir="$CONFDIR" \
  --set connection_strategy=lazy \
  -s /opt/janissary/janissary_addon.py
```

---

## 10. Startup and Health

### Startup sequence

```
1. Divan starts (network_mode: host, port 8600)

   └── healthcheck: GET /health → 200

2. Janissary starts (depends_on: divan healthy)
   ├── janissary-entrypoint.sh:
   │   ├── Prepare mitmproxy CA confdir
   │   ├── Poll Divan /health until 200 (curl loop, 1s interval, 60s timeout)
   │   ├── Start appeal API (uvicorn, port 8081, background)
   │   └── Start mitmdump (port 8080, foreground)
   │
   ├── JanissaryAddon.running():
   │   └── Start DivanPoller background thread
   │       └── First poll: GET /janissary/state
   │           ├── Success → has_loaded = True, traffic flows
   │           └── Failure → has_loaded = False, all traffic blocked (fail-closed)
   │
   └── healthcheck: GET /health on appeal API → 200
       (only returns 200 after has_loaded = True)

3. Sentinel starts (host networking, not through Janissary)

4. Vizier starts (depends_on: janissary healthy, through Janissary)

5. Provinces start on demand (through Janissary)
```

### Health endpoint

The appeal API server exposes a health endpoint:

```
GET /health
```

**Response (200)** -- Janissary is ready to serve traffic:
```json
{
  "status": "ok",
  "has_state": true,
  "last_poll_at": "2026-04-12T10:30:05Z",
  "cache_age_seconds": 3
}
```

**Response (503)** -- Janissary is not ready:
```json
{
  "status": "initializing",
  "has_state": false,
  "last_poll_at": null,
  "cache_age_seconds": null
}
```

The healthcheck returns 503 until the first successful Divan poll. This
prevents Docker from routing dependent services (Vizier) until Janissary
has loaded its rules.

### Graceful shutdown

On `SIGTERM` (Docker stop):

1. mitmproxy stops accepting new connections
2. In-flight requests complete (mitmproxy default: 5s drain)
3. DivanPoller thread stops (`_running = False`)
4. Appeal API server shuts down (uvicorn handles SIGTERM)
5. Process exits

No state persistence needed -- all state is in Divan. On restart,
Janissary re-polls and rebuilds the cache.

### Failure modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| Divan down at Janissary start | Janissary blocks all traffic (fail-closed) | Auto-recovers when Divan comes up (poller retries every 5s) |
| Divan goes down after start | Janissary uses cached state | Auto-recovers on next successful poll |
| Janissary crashes | All province/Vizier traffic fails (no proxy) | Docker `restart: unless-stopped` restarts it. Sultan has SSH fallback. |
| Appeal API crashes | Appeals fail, proxy still works | Restart container. Traffic rules unaffected. |
| mitmproxy crashes | All traffic blocked (proxy gone) | Docker restart. Fail-closed by design. |
