# Janissary Technical Specification

> Egress proxy for the Sultanate platform. For requirements see
> [JANISSARY_MVP_PRD.md](JANISSARY_MVP_PRD.md). For the content-inspector
> sibling, see [KASHIF_MVP_PRD.md](KASHIF_MVP_PRD.md). For shared state
> contract see [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md). For architecture
> context see [SULTANATE_MVP.md](SULTANATE_MVP.md).

---

## 1. Sandcat Fork Approach

Janissary is a **fork** of [Sandcat](https://github.com/VirtusLab/sandcat)
(VirtusLab, Apache 2.0 license). Sandcat is a mitmproxy-based dev container
sandbox that transparently intercepts all container traffic via WireGuard.
Janissary extends it with multi-province support, Divan-driven dynamic
configuration, credential injection (lease-aware), Kashif integration for
appeal triage, and audit logging to Divan.

### What Janissary keeps from Sandcat

| Sandcat feature | Janissary usage |
|-----------------|-----------------|
| WireGuard transparent proxy (`wg-client` + iptables) | Kept as-is. Province traffic routed through mitmproxy without HTTP_PROXY env vars. |
| Full MITM on all HTTPS (CA cert trusted by containers) | Kept as-is. All TLS connections are decrypted and inspected. |
| mitmproxy addon pattern (`request()` hook, class structure) | `JanissaryAddon` class inherits the addon pattern from `SandcatAddon`. |
| CA certificate generation and distribution | Kept. Sultanate CA generated at deploy time, mounted into all province containers. |
| iptables kill-switch (fail-closed when WireGuard down) | Kept as-is. If tunnel drops, all province egress is dropped. |
| Docker Compose orchestration (`wg-client` + `mitmproxy` containers) | Kept. Extended with Divan, Kashif, per-province wg-client sidecars. |

### What Janissary replaces in the fork

| Sandcat feature | Janissary replacement |
|-----------------|-----------------------|
| File-based `settings.json` (read once at startup) | Divan API polling (5s interval, background thread via `DivanPoller`) |
| Single agent container | Multi-province: source IP awareness, per-province traffic rules and grants |
| `_substitute_secrets()` placeholder replacement | `_inject_credentials()` direct HTTP header injection (agent never sees credential; lease-aware fail-closed skip on expired lease) |
| `fnmatch` glob domain matching | Exact string equality (case-insensitive, trailing dot stripped) |
| `dns_request()` hook for DNS filtering | Not needed (WireGuard handles DNS routing) |
| `load(loader)` reads settings once | `running()` starts `DivanPoller` background thread |
| No appeal system | FastAPI appeal API on port 8081, forwards to Kashif for triage |
| No audit logging | JSON audit entries posted to Divan `/audit` (canonical) + optional local JSONL for recovery |
| Static config (read once) | Dynamic polling with fail-closed on fresh start |

### Fork file structure

```
janissary/
├── janissary_addon.py    # mitmproxy addon (forked from SandcatAddon)
│                         #   traffic rules, credential injection (lease-aware),
│                         #   audit writes to Divan
├── janissary_api.py      # FastAPI appeal/access-request HTTP API (port 8081)
├── janissary_kashif.py   # Kashif client (POST /screen/appeal, /screen/ingress)
├── janissary_state.py    # DivanPoller + JanissaryStateCache
├── janissary_audit.py    # AuditClient (POST /audit to Divan)
├── janissary_config.py   # Config loading (config.yaml)
├── janissary_main.py     # Process entry point (starts mitmproxy + API server)
├── wg-client/
│   ├── Dockerfile        # WireGuard + iptables kill-switch container
│   └── entrypoint.sh     # WireGuard interface setup + iptables rules
├── Dockerfile            # mitmproxy + addon container
├── janissary-entrypoint.sh
├── config.yaml           # Default config template
└── requirements.txt      # mitmproxy, fastapi, uvicorn, httpx, pyyaml
```

### License

Janissary retains Sandcat's Apache 2.0 license and includes attribution per
the license terms. All modifications are clearly documented in the fork's
commit history.

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

### Full MITM on all HTTPS

All HTTPS traffic is MITM'd by default. mitmproxy terminates every TLS
connection using the Sultanate CA, decrypts the request, applies traffic
rules and credential injection, then opens a new TLS connection to the
upstream server. This means all 4 traffic rules apply uniformly to both
HTTP and HTTPS requests.

### Cert-pinning escape hatch

Some services (e.g., certain SDKs or clients that pin certificates) break
under MITM. For these, domains can be added to a `passthrough_domains` list
in `config.yaml`:

```yaml
passthrough_domains:
  - "example-pinned-service.com"
  - "another-pinned.io"
```

Passthrough domains skip MITM: mitmproxy tunnels the raw TLS connection
without decryption. Traffic rules still apply at the CONNECT level
(blacklist blocks the CONNECT; whitelist allows the tunnel; non-whitelisted
read-only/write-block rules **cannot be enforced** since the HTTP method is
inside the encrypted tunnel). This is an opt-in escape hatch for
compatibility, not the default behavior.

```python
def tls_clienthello(self, flow: tls.ClientHelloData):
    """Skip MITM for cert-pinning domains listed in passthrough_domains."""
    host = flow.context.server.address[0] if flow.context.server else None
    if host and self._normalize_domain(host) in self.passthrough_domains:
        flow.ignore_connection = True
```

### mitmproxy CA configuration

mitmproxy is started with `--set confdir=/opt/mitmproxy` where the CA files
are pre-placed. mitmproxy expects:

```
/opt/mitmproxy/
├── mitmproxy-ca.pem           # combined key + cert
└── mitmproxy-ca-cert.pem      # cert only
```

The Janissary entrypoint converts the PEM cert+key into the formats
mitmproxy expects:

```bash
# janissary-entrypoint.sh (CA setup excerpt)
CONFDIR="/opt/mitmproxy"
mkdir -p "$CONFDIR"
cat /certs/sultanate-ca.key /certs/sultanate-ca.pem > "$CONFDIR/mitmproxy-ca.pem"
cp /certs/sultanate-ca.pem "$CONFDIR/mitmproxy-ca-cert.pem"
```

---

## 3. Proxy Configuration

### WireGuard transparent proxy architecture

Janissary uses Sandcat's WireGuard transparent proxy approach. Province
containers do **not** set `HTTP_PROXY` / `HTTPS_PROXY` env vars. Instead,
all traffic is transparently routed through mitmproxy via WireGuard tunnels
and iptables NAT rules.

```
Province container
  │  (shares network namespace with wg-client sidecar)
  │
  ├── all outbound traffic → wg0 interface
  │
  └── iptables NAT redirect:
        port 80  → mitmproxy 8080
        port 443 → mitmproxy 8080
        other    → dropped (kill-switch)

wg-client sidecar
  │  WireGuard tunnel → Janissary mitmproxy container
  │
  └── iptables kill-switch:
        -A OUTPUT -o wg0 -j ACCEPT
        -A OUTPUT -d 127.0.0.0/8 -j ACCEPT
        -A OUTPUT -j DROP
```

### Listen addresses and ports

| Service | Address | Port | Protocol | Purpose |
|---------|---------|------|----------|---------|
| mitmproxy | `0.0.0.0` | `8080` | HTTP (transparent mode) | Intercepts all province/Vizier HTTP and HTTPS traffic |
| Appeal API | `0.0.0.0` | `8081` | HTTP | Appeal and access-request endpoints |
| Divan | `127.0.0.1` | `8600` | HTTP | Shared state store (Janissary polls this) |
| Kashif | `127.0.0.1` | `8082` | HTTP | Content inspector (Janissary forwards appeals to this) |

mitmproxy runs in **transparent mode** (`--mode transparent`), not regular
forward proxy mode. Traffic arrives via iptables NAT redirect, not via
explicit proxy configuration.

### WireGuard wg-client setup

Each province gets a `wg-client` sidecar container. The province container
shares the sidecar's network namespace
(`network_mode: "service:wg-client-{id}"`), so all province traffic flows
through the sidecar's network stack.

The wg-client sidecar:
1. Establishes a WireGuard tunnel to the Janissary container
2. Configures iptables to redirect ports 80 and 443 to the mitmproxy
   endpoint
3. Installs a kill-switch: if the WireGuard tunnel goes down, all traffic
   is dropped (fail-closed)

```bash
#!/bin/bash
# wg-client/entrypoint.sh
set -euo pipefail

# Configure WireGuard interface
wg-quick up /etc/wireguard/wg0.conf

# NAT redirect: route HTTP/HTTPS to mitmproxy
iptables -t nat -A OUTPUT -p tcp --dport 80  -j DNAT --to-destination ${MITMPROXY_HOST}:8080
iptables -t nat -A OUTPUT -p tcp --dport 443 -j DNAT --to-destination ${MITMPROXY_HOST}:8080

# Kill-switch: only allow traffic through WireGuard tunnel
iptables -A OUTPUT -o wg0 -j ACCEPT
iptables -A OUTPUT -d 127.0.0.0/8 -j ACCEPT
iptables -A OUTPUT -j DROP

# Keep container running
exec sleep infinity
```

### Network topology

```
Docker host
├── Janissary container (network_mode: host)
│   ├── mitmproxy on 0.0.0.0:8080 (transparent proxy)
│   ├── Appeal API on 0.0.0.0:8081
│   ├── WireGuard server interface (wg0)
│   └── Internet egress via host network
│
├── Kashif container (network_mode: host)
│   └── FastAPI on 127.0.0.1:8082 (screen/appeal, screen/ingress)
│
├── Divan container (network_mode: host)
│   ├── 127.0.0.1:8600 (API, readable by all Sultanate containers)
│   └── 127.0.0.1:8601 (dashboard, SSH-tunnel only)
│
├── wg-client-prov-a1b2c3 (cap_add: NET_ADMIN)
│   ├── WireGuard client tunnel → Janissary wg0
│   ├── iptables NAT redirect (80,443 → mitmproxy)
│   ├── iptables kill-switch
│   └── Shared network namespace with province container
│
├── province-prov-a1b2c3 (network_mode: "service:wg-client-prov-a1b2c3")
│   ├── All traffic goes through wg-client's network stack
│   ├── CA cert mounted at /usr/local/share/ca-certificates/sultanate-ca.crt
│   └── No HTTP_PROXY/HTTPS_PROXY env vars needed
│
└── (more wg-client + province pairs as needed)
```

Janissary runs with `network_mode: host` to get both internet egress and
localhost access to Divan (8600) and Kashif (8082). Province containers
have no direct internet route -- their only path to the internet is
through the WireGuard tunnel to Janissary's mitmproxy.

### Province source IP identification

Each WireGuard peer (wg-client sidecar) has a unique IP address within the
WireGuard subnet. Janissary identifies the source province by the
WireGuard peer IP of the incoming connection. This IP is registered in
Divan when Vizier creates the province.

### Janissary configuration file

`/opt/sultanate/janissary/config.yaml`:

```yaml
# Janissary configuration
proxy:
  listen_host: "0.0.0.0"
  listen_port: 8080
  mode: "transparent"

api:
  listen_host: "0.0.0.0"
  listen_port: 8081

divan:
  url: "http://127.0.0.1:8600"
  api_key_env: "DIVAN_KEY_JANISSARY"     # reads from environment
  poll_interval_seconds: 5
  health_check_interval_seconds: 2
  health_check_timeout_seconds: 60

kashif:
  url: "http://127.0.0.1:8082"
  timeout_seconds: 5                     # Kashif hard timeout
  escalate_on_timeout: true              # write kashif_verdict=escalate
                                         # if Kashif doesn't respond

ca:
  cert_path: "/certs/sultanate-ca.pem"
  key_path: "/certs/sultanate-ca.key"
  confdir: "/opt/mitmproxy"

wireguard:
  interface: "wg0"
  listen_port: 51820
  subnet: "10.13.13.0/24"                # WireGuard peer subnet
  server_address: "10.13.13.1/24"

appeal:
  one_time_timeout_minutes: 5            # configurable, default 5

passthrough_domains: []                  # cert-pinning escape hatch

logging:
  level: "INFO"                          # DEBUG, INFO, WARNING, ERROR
  local_audit_file: "/var/log/janissary/audit.jsonl"
                                         # optional local mirror of Divan
                                         # audit (for recovery / debugging);
                                         # Divan /audit is canonical
```

### mitmproxy launch command

```bash
mitmdump \
  --mode transparent \
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

Evaluated per-request in the `request()` mitmproxy hook. All 4 rules apply
uniformly to HTTP and HTTPS. mitmproxy decrypts all TLS connections using
the Sultanate CA, so every request arrives at the `request()` hook with
full method, host, path, and headers visible.

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
    # Check one-time approved appeals first (includes Kashif auto-allow)
    if self.state.has_approved_appeal(source_ip, method, host):
        return "allow"

    return "block_write"
```

### HTTPS handling

All HTTPS traffic is MITM'd by default. When a client opens a TLS
connection (CONNECT), mitmproxy:

1. Terminates TLS with the Sultanate CA cert
2. Decrypts the request
3. The `request()` hook fires with full request details
4. Traffic rules and credential injection are applied
5. mitmproxy opens a new TLS connection to the upstream server
6. Response flows back to the client

Blacklisted domains are also checked at the CONNECT level for early
rejection:

```python
def http_connect(self, flow: http.HTTPFlow):
    """Early blacklist rejection at CONNECT time."""
    host = self._normalize_domain(flow.request.pretty_host)

    # Blacklisted -> block immediately (respond 403 to CONNECT)
    if self.state.has_loaded and host in self.state.blacklist:
        flow.response = http.Response.make(
            403,
            json.dumps({
                "error": "blacklisted",
                "message": f"Domain {host} is on the global blacklist.",
            }),
            {"Content-Type": "application/json"},
        )
```

### Domain matching semantics

**Exact string equality on the domain label.** No glob patterns, no
wildcard subdomain matching, no regex.

```
Whitelist contains: "github.com"

github.com       -> match
api.github.com   -> no match  (must be listed separately)
www.github.com   -> no match
GITHUB.COM       -> match     (case-insensitive comparison)
github.com.      -> match     (trailing dot stripped before comparison)
```

Implementation:

```python
def _normalize_domain(self, domain: str) -> str:
    return domain.lower().rstrip(".")
```

All domain comparisons use normalized forms. Divan stores domains in
lowercase without trailing dots.

### Audit logging

Every request decision is posted to Divan's `/audit` endpoint as a JSON
entry. Format:

```json
{
  "component": "janissary",
  "severity": "info",
  "province_id": "prov-a1b2c3",
  "source_ip": "10.13.13.5",
  "action": "http_request",
  "verdict": "allow",
  "rule": "whitelist",
  "detail": {
    "method": "POST",
    "host": "api.github.com",
    "path": "/repos/stranma/EFM/pulls",
    "mitm": true,
    "credential_injected": true,
    "grant_id": "grant-x1y2z3",
    "lease_expired": false
  }
}
```

`severity` selection:
- `info` -- normal allow decisions (whitelist, read-only, approved appeal)
- `alert` -- blocks (blacklist, write_block), lease-expired skip,
  repeated blacklist hits
- `error` -- Divan write failures, Kashif unreachable

A local JSONL mirror at `/var/log/janissary/audit.jsonl` is also written
for recovery/debugging when Divan is unreachable. If Divan is down, local
file becomes the only sink until the DivanPoller reconnects and flushes
buffered entries.

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
@dataclass
class Grant:
    header: str
    value: str
    openbao_lease_id: str | None = None
    lease_expires_at: datetime | None = None   # parsed from ISO 8601

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
    # Includes Kashif auto-allow approvals and Sultan-approved one-time decisions.
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

        # Grants: index by (source_ip, domain) with lease info
        cache.grants = {}
        for g in data["grants"]:
            key = (g["source_ip"], g["match"]["domain"].lower().rstrip("."))
            lease_expires_at = None
            if g.get("lease_expires_at"):
                lease_expires_at = datetime.fromisoformat(
                    g["lease_expires_at"].replace("Z", "+00:00")
                )
            cache.grants[key] = Grant(
                header=g["inject"]["header"],
                value=g["inject"]["value"],
                openbao_lease_id=g.get("openbao_lease_id"),
                lease_expires_at=lease_expires_at,
            )

        # Approved appeals (one-time, within configurable timeout window)
        cache.approved_appeals = {}
        for a in data["approved_appeals"]:
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
    Divan's bulk state endpoint already filters by the configured timeout window."""
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
| Fresh start, never polled Divan | **Block all traffic.** `has_loaded = False` -- every request returns 503 with "Janissary initializing, waiting for state" |
| Divan unreachable after successful poll | **Use cached state.** Last-known rules remain in effect. Log warnings. |
| Divan returns error (500, etc.) | Same as unreachable -- keep cached state, log warning |
| Province IP not in cache | Traffic from unknown IPs is blocked. Only registered, running provinces are served. |

### Polling interval

Default: **5 seconds** (configurable via `config.yaml`). This means:

- Whitelist additions take up to 5s to take effect
- One-time appeal approvals are available within 5s of Sultan's decision
- Blacklist changes propagate within 5s
- Grant rotations (Aga renews an OpenBao lease and rewrites the value)
  take up to 5s to be reflected in what Janissary injects

---

## 6. Credential Injection

### How it works

All HTTPS is MITM'd by default, so every request arrives at the `request()`
hook fully decrypted. Credential injection is straightforward: look up a
grant for the source IP and domain, **verify the grant's OpenBao lease is
still valid**, and inject the header.

```python
def _inject_credentials(self, flow: http.HTTPFlow, source_ip: str) -> dict:
    """Returns audit detail dict describing the injection decision."""
    host = self._normalize_domain(flow.request.pretty_host)
    grant = self.state.get_grant(source_ip, host)
    if grant is None:
        return {"credential_injected": False, "grant_id": None}

    # Lease expiry check (Aga-driven renewal happens async; if we see an
    # expired lease here, Aga hasn't renewed yet -- fail closed).
    if grant.lease_expires_at is not None:
        now = datetime.now(timezone.utc)
        if grant.lease_expires_at <= now:
            # Skip injection. Do NOT block the request: upstream will return
            # 401/403 naturally, signalling to the agent that the token is
            # stale. Aga sees our audit entry and re-issues.
            return {
                "credential_injected": False,
                "grant_id": grant.openbao_lease_id,
                "lease_expired": True,
                "lease_expires_at": grant.lease_expires_at.isoformat(),
            }

    # Add or replace the header specified in the grant
    flow.request.headers[grant.header] = grant.value
    return {
        "credential_injected": True,
        "grant_id": grant.openbao_lease_id,
        "lease_expired": False,
    }
```

Grants where `lease_expires_at` is `None` (KV-mode grants for Sultan-
provided tokens with no dynamic lease) bypass the expiry check and inject
unconditionally. See `SULTANATE_MVP.md` credential model for KV vs
dynamic mode.

### Request flow

```
Request arrives (already decrypted by mitmproxy MITM)
  -> Look up grant for (source_ip, domain)
  -> If grant.lease_expires_at <= now:
       -> skip injection, write audit severity=alert, continue
  -> Else: set/replace the specified header
  -> Forward to upstream
```

No separate "MITM decision" is needed -- all connections are MITM'd, so
credential injection applies uniformly.

### Grant matching

Grant lookup uses exact match on `(source_ip, normalized_domain)`:

```
Grant: source_ip=10.13.13.5, domain=api.github.com
Request from 10.13.13.5 to api.github.com  -> injected (if lease valid)
Request from 10.13.13.5 to github.com      -> NOT injected (different domain)
Request from 10.13.13.6 to api.github.com  -> NOT injected (different source)
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

If the lease has expired at injection time, the province's request goes
without the header and receives a `401` / `403` from upstream. The agent
perceives this as "my credentials are broken" and may surface the error
or retry; Aga sees the audit entry and re-issues the grant.

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
  "payload": "<full request body, forwarded to Kashif>",
  "justification": "Need to submit the PR review comment for task #42"
}
```

Janissary identifies the caller by source IP (from the TCP connection).

**Processing:**

1. Extract `source_ip` from the request's TCP connection.
2. Look up `province_id` from the state cache.
3. Write to Divan: `POST /appeals` with `source_ip`, `province_id`, `url`,
   `method`, `justification`. Receive `appeal_id`.
4. **Forward to Kashif:** `POST http://127.0.0.1:8082/screen/appeal` with
   `{ appeal_id, payload, justification, url, method, province_id }`.
   Wait for Kashif's response with `timeout=5s` (configurable).
5. If Kashif responds within timeout: verdict is written to Divan by
   Kashif itself via `PATCH /appeals/{appeal_id}/kashif_verdict`. Divan
   auto-transitions on `allow` (approved, one-time) or `block` (denied).
6. If Kashif times out or returns 5xx: Janissary writes
   `PATCH /appeals/{appeal_id}/kashif_verdict` with
   `kashif_verdict="escalate"` and `detail.kashif_timeout=true`. The
   appeal stays pending; Vizier relays to Sultan.
7. Return response to the agent.

**Response (202 — accepted):**

```json
{
  "status": "pending",
  "appeal_id": "appeal-m1n2o3",
  "message": "Appeal submitted. Kashif is screening the content; if unresolved in a few seconds it will be escalated to Sultan."
}
```

**Response (400):** missing or invalid fields.
**Response (503):** Divan unreachable.

Janissary's HTTP response to the agent is immediate (202) -- the agent
should poll or retry later. Kashif's verdict and Sultan's decision (if
escalated) arrive asynchronously and are reflected in Divan's state on
the next Janissary poll.

#### `POST /api/request_access`

Request new credentials or permanent whitelist addition.

**Request:**
```json
{
  "service": "api.example.com",
  "scope": "repo:write for stranma/EFM",
  "justification": "Need push access to submit PRs"
}
```

**Processing:**

1. Extract `source_ip`, look up `province_id`.
2. Forward the `{service, scope, justification}` text to Kashif via
   `POST /screen/ingress` (screens the justification before it reaches
   Aga's LLM context). Wait for Kashif verdict.
3. If Kashif=block: respond `403 { status: "denied", reason: "blocked by Kashif" }`.
   Audit entry severity=alert. Sultan+Aga notified via polling (see
   SULTANATE_MVP.md appeal flow).
4. If Kashif=allow or escalate: write to Divan
   (`POST /access_requests` -- a variant of appeals, or overload
   `/appeals` with a special URL scheme `access-request://{service}`).
5. Vizier picks it up and routes to Sultan via Telegram.

**Response (202):**

```json
{
  "status": "pending",
  "message": "Access request submitted. Kashif screened; awaiting Sultan's decision via Vizier."
}
```

### Province access to appeal API

Since provinces share the wg-client sidecar's network namespace, the
appeal API is accessible via the Janissary host at port 8081. The firman
configures the appeal API address as an environment variable:

```bash
JANISSARY_API="http://${JANISSARY_HOST}:8081"
```

The MCP tool in provinces (provided by the firman) calls the API:

```python
# In province MCP tool implementation
import httpx

JANISSARY_API = os.environ["JANISSARY_API"]

def appeal_request(url: str, method: str, payload: str, justification: str) -> dict:
    resp = httpx.post(f"{JANISSARY_API}/api/appeal", json={
        "url": url,
        "method": method,
        "payload": payload,
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
1. Agent POSTs to api.example.com -> blocked (rule 4) -> gets 403
2. Agent calls appeal_request(url, "POST", payload, justification)
3. Janissary writes appeal to Divan (status: pending) AND forwards
   payload + justification to Kashif /screen/appeal
4. Kashif returns verdict within ~2s (typical). Three branches:
   a. Kashif "allow" -> Divan auto-approves as one-time
   b. Kashif "block" -> Divan auto-denies, severity=alert, notify
   c. Kashif "escalate" or timeout -> appeal stays pending
5. If (c): Vizier polls Divan, relays to Sultan via Telegram
6. Sultan approves (one-time) via Vizier -> Vizier PATCH /appeals
7. Janissary's next poll picks up the approval in approved_appeals
8. Agent retries POST to api.example.com
9. Janissary checks _evaluate_traffic_rules():
   - Not blacklisted
   - Not whitelisted -> check read-only -> POST is not GET/HEAD
     -> check appeals
   - has_approved_appeal(source_ip, "POST", "api.example.com") -> True
   - Result: allow
10. Request goes through (one time only)
```

### One-time approval timeout

The one-time approval timeout is **configurable** via `config.yaml`:

```yaml
appeal:
  one_time_timeout_minutes: 5    # default: 5 minutes
```

Divan's bulk state endpoint (`GET /janissary/state`) filters
`approved_appeals` to only include `one-time` approvals with `resolved_at`
within this configured window. After the timeout expires, the approval
is no longer returned and the agent must appeal again.

---

## 8. Kashif Integration

Janissary forwards two kinds of content to Kashif:

1. **Appeal payloads** (blocked write request + justification). Endpoint
   `POST http://127.0.0.1:8082/screen/appeal`. See Appeal Mechanics
   above.
2. **Access-request justifications.** Endpoint `POST /screen/ingress`.
   Screens the free-form text before Aga sees it in its LLM context.

### Client code

```python
# janissary_kashif.py

class KashifClient:
    def __init__(self, config: JanissaryConfig):
        self.url = config.kashif.url                 # http://127.0.0.1:8082
        self.timeout = config.kashif.timeout_seconds # 5

    def screen_appeal(self, appeal_id, payload, justification,
                       url, method, province_id) -> dict:
        try:
            resp = httpx.post(
                f"{self.url}/screen/appeal",
                json={
                    "appeal_id": appeal_id,
                    "payload": payload,
                    "justification": justification,
                    "url": url,
                    "method": method,
                    "province_id": province_id,
                },
                timeout=self.timeout,
            )
            resp.raise_for_status()
            return resp.json()    # { verdict: "allow"|"block"|"escalate", ... }
        except Exception as e:
            logger.warning(f"Kashif unreachable or timed out: {e}")
            return {"verdict": "escalate", "reason": "kashif_timeout"}

    def screen_ingress(self, content: str, source: str,
                        province_id: str) -> dict:
        try:
            resp = httpx.post(
                f"{self.url}/screen/ingress",
                json={"content": content, "source": source,
                      "province_id": province_id},
                timeout=self.timeout,
            )
            resp.raise_for_status()
            return resp.json()
        except Exception as e:
            logger.warning(f"Kashif unreachable or timed out: {e}")
            return {"verdict": "escalate", "reason": "kashif_timeout"}
```

### Fail-closed behavior

If Kashif is down or times out, Janissary treats the verdict as
`escalate`. The appeal or access request stays pending for Sultan review;
**Janissary never auto-approves on Kashif failure**. An audit entry with
`severity=error` is written for operator visibility.

---

## 9. 403 Response Body

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
    "source_ip": "10.13.13.5",
    "province_id": "prov-a1b2c3",
    "rule": "write_block"
  },
  "appeal": {
    "url": "http://${JANISSARY_HOST}:8081/api/appeal",
    "method": "POST",
    "content_type": "application/json",
    "body_template": {
      "url": "https://api.example.com/data",
      "method": "POST",
      "payload": "<full request body>",
      "justification": "<explain why you need this>"
    },
    "instructions": "Submit an appeal by POSTing to the appeal URL above. Include the full original payload and a justification explaining why this write is needed. Kashif will triage; if unresolved Sultan will decide. After approval, retry your original request."
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
    "source_ip": "10.13.13.5",
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

## 10. Docker Deployment

### docker-compose.yml

```yaml
services:
  openbao:
    image: openbao/openbao:2.5.3
    volumes:
      - openbao-data:/openbao/data
      - /opt/sultanate/openbao/config.hcl:/openbao/config/config.hcl:ro
    network_mode: host             # 127.0.0.1:8200 only
    cap_drop: [ALL]
    cap_add: [IPC_LOCK]
    security_opt: ["no-new-privileges:true"]
    ulimits: { core: 0 }
    restart: unless-stopped
    # Manual unseal by Sultan at first boot; auto-unseal deferred to Phase 2

  divan:
    build:
      context: ./divan
    depends_on:
      openbao:
        condition: service_started
    volumes:
      - /opt/sultanate/divan.db:/opt/sultanate/divan.db
      - /opt/sultanate/divan.env:/opt/sultanate/divan.env:ro
      - /opt/sultanate/dashboard.env:/opt/sultanate/dashboard.env:ro
    network_mode: host             # 127.0.0.1:8600 + 127.0.0.1:8601
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
      kashif:
        condition: service_healthy
    network_mode: host             # internet egress + localhost Divan/Kashif access
    volumes:
      - /opt/sultanate/certs:/certs:ro
      - /opt/sultanate/janissary/config.yaml:/opt/janissary/config.yaml:ro
      - /opt/sultanate/divan.env:/opt/sultanate/divan.env:ro
      - janissary-logs:/var/log/janissary
    env_file:
      - /opt/sultanate/divan.env    # provides DIVAN_KEY_JANISSARY
    cap_add:
      - NET_ADMIN                   # WireGuard server interface
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python3", "-c",
        "import httpx; r=httpx.get('http://127.0.0.1:8081/health'); r.raise_for_status()"]
      interval: 5s
      timeout: 3s
      retries: 10

  kashif:
    build:
      context: ./kashif
    depends_on:
      divan:
        condition: service_healthy
    network_mode: host             # 127.0.0.1:8082
    volumes:
      - /opt/sultanate/divan.env:/opt/sultanate/divan.env:ro
      - kashif-models:/opt/kashif/models    # pre-baked model weights
    env_file:
      - /opt/sultanate/divan.env    # provides DIVAN_KEY_KASHIF
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://127.0.0.1:8082/health"]
      interval: 5s
      timeout: 3s
      retries: 30                   # slower cold start while models load

  aga:
    build:
      context: ./aga
    depends_on:
      openbao:
        condition: service_started
      divan:
        condition: service_healthy
      janissary:
        condition: service_healthy
      kashif:
        condition: service_healthy
    network_mode: host             # direct host networking
    volumes:
      - /opt/sultanate/aga/state:/opt/aga/state
      - /opt/sultanate/aga/bootstrap:/opt/aga/bootstrap:ro
      - /opt/sultanate/divan.env:/opt/sultanate/divan.env:ro
      - /opt/sultanate/aga/openbao.env:/opt/aga/openbao.env:ro
    env_file:
      - /opt/sultanate/divan.env
      - /opt/sultanate/aga/openbao.env
    cap_add: [NET_ADMIN]            # iptables for port_request fulfilment
    restart: unless-stopped

volumes:
  openbao-data:
  janissary-logs:
  kashif-models:
```

### WireGuard province networking

Province containers are created dynamically by Vizier. Each province gets
a `wg-client` sidecar container, and the province container shares the
sidecar's network namespace.

Vizier creates the wg-client sidecar and province as a pair:

```yaml
# Created dynamically by Vizier for each province
# (conceptual docker-compose fragment, actual creation via Docker API)

services:
  wg-client-prov-a1b2c3:
    build: ./janissary/wg-client
    cap_add:
      - NET_ADMIN
    volumes:
      - /opt/sultanate/provinces/prov-a1b2c3/wg0.conf:/etc/wireguard/wg0.conf:ro
    environment:
      - MITMPROXY_HOST=10.13.13.1    # Janissary's WireGuard address
    restart: unless-stopped

  province-prov-a1b2c3:
    image: openclaw/openclaw:vYYYY.M.D
    network_mode: "service:wg-client-prov-a1b2c3"
    volumes:
      - /opt/sultanate/provinces/prov-a1b2c3/data:/opt/data
      - /opt/sultanate/certs/sultanate-ca.pem:/usr/local/share/ca-certificates/sultanate-ca.crt:ro
    environment:
      - JANISSARY_API=http://10.13.13.1:8081
      # No HTTP_PROXY / HTTPS_PROXY -- traffic is transparently intercepted
    depends_on:
      - wg-client-prov-a1b2c3
```

### WireGuard configuration

Each province gets a unique WireGuard peer configuration. Vizier generates
these at province creation time.

**Janissary server config** (`/opt/sultanate/janissary/wg0.conf`):

```ini
[Interface]
Address = 10.13.13.1/24
ListenPort = 51820
PrivateKey = <janissary-private-key>

# Peers added dynamically by Vizier
[Peer]
# prov-a1b2c3
PublicKey = <province-public-key>
AllowedIPs = 10.13.13.5/32

[Peer]
# prov-d4e5f6
PublicKey = <province-public-key>
AllowedIPs = 10.13.13.6/32
```

**Province wg-client config**
(`/opt/sultanate/provinces/prov-a1b2c3/wg0.conf`):

```ini
[Interface]
Address = 10.13.13.5/32
PrivateKey = <province-private-key>

[Peer]
PublicKey = <janissary-public-key>
Endpoint = <host-ip>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

`AllowedIPs = 0.0.0.0/0` routes all traffic through the WireGuard tunnel
to Janissary.

### Janissary Dockerfile

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    wireguard-tools \
    iptables \
    curl \
  && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir \
    mitmproxy==11.* \
    fastapi==0.115.* \
    uvicorn[standard]==0.34.* \
    httpx==0.28.* \
    pyyaml==6.*

COPY . /opt/janissary/
WORKDIR /opt/janissary

RUN mkdir -p /opt/mitmproxy /var/log/janissary
RUN chmod +x /opt/janissary/janissary-entrypoint.sh

EXPOSE 8080 8081 51820/udp

ENTRYPOINT ["/opt/janissary/janissary-entrypoint.sh"]
```

### wg-client Dockerfile

```dockerfile
FROM alpine:3.20

RUN apk add --no-cache wireguard-tools iptables bash

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
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

# Set up WireGuard server interface
wg-quick up /opt/janissary/wg0.conf || {
  echo "FATAL: WireGuard setup failed. Exiting."
  exit 1
}
echo "WireGuard interface up."

# Wait for Divan to be healthy
echo "Waiting for Divan..."
until curl -sf http://127.0.0.1:8600/health > /dev/null 2>&1; do
  sleep 1
done
echo "Divan is ready."

# Wait for Kashif to be healthy
echo "Waiting for Kashif..."
until curl -sf http://127.0.0.1:8082/health > /dev/null 2>&1; do
  sleep 2
done
echo "Kashif is ready."

# Start appeal API in background
python3 /opt/janissary/janissary_api.py &
API_PID=$!

# Start mitmproxy (foreground, transparent mode)
exec mitmdump \
  --mode transparent \
  --listen-host 0.0.0.0 \
  --listen-port 8080 \
  --set confdir="$CONFDIR" \
  --set connection_strategy=lazy \
  -s /opt/janissary/janissary_addon.py
```

---

## 11. Startup and Health

### Startup sequence

```
1. OpenBao starts (network_mode: host, 127.0.0.1:8200)
   +-- Manual unseal by Sultan (first boot; subsequent boots still manual
       in MVP, auto-unseal deferred to Phase 2)

2. Divan starts (network_mode: host, port 8600 + dashboard 8601)
   +-- healthcheck: GET /health -> 200

3. Janissary starts (depends_on: divan healthy, kashif healthy)
   +-- janissary-entrypoint.sh:
   |   +-- Prepare mitmproxy CA confdir
   |   +-- Start WireGuard server interface (wg-quick up)
   |   |   +-- Success -> continue
   |   |   +-- Failure -> exit 1 (container fails, Docker restarts)
   |   +-- Poll Divan /health until 200 (curl loop, 1s interval, 60s timeout)
   |   +-- Poll Kashif /health until 200 (curl loop, 2s interval)
   |   +-- Start appeal API (uvicorn, port 8081, background)
   |   +-- Start mitmdump (port 8080, foreground, transparent mode)
   |
   +-- JanissaryAddon.running():
   |   +-- Start DivanPoller background thread
   |       +-- First poll: GET /janissary/state
   |           +-- Success -> has_loaded = True, traffic flows
   |           +-- Failure -> has_loaded = False, all traffic blocked (fail-closed)
   |
   +-- healthcheck: GET /health on appeal API -> 200
       (only returns 200 after has_loaded = True)

4. Kashif starts (parallel to Janissary; models load over ~10-30s)
   +-- healthcheck: GET /health -> 200 after all three models resident

5. Aga starts (depends_on: openbao, divan, janissary, kashif all healthy)
   +-- Authenticates to OpenBao via AppRole from /opt/aga/openbao.env
   +-- Reads GitHub App private key from OpenBao KV
   +-- Starts Divan poller (looks for new provinces in status=creating)

6. Vizier starts (depends_on: janissary healthy, through Janissary)
   +-- Vizier's wg-client sidecar connects to Janissary's WireGuard

7. Provinces start on demand (Vizier creates wg-client + province pairs)
   +-- wg-client: WireGuard tunnel + iptables kill-switch
   +-- Province: shares wg-client network, CA cert installed
```

### WireGuard health in the startup chain

The WireGuard server interface must be established before mitmproxy
accepts traffic. The entrypoint enforces this ordering: `wg-quick up`
runs before `mitmdump` starts. If WireGuard setup fails, the entrypoint
exits with a non-zero code and Docker restarts the container.

For province wg-client sidecars, the WireGuard tunnel must be up before
the province container starts. The `depends_on` relationship in the
dynamic container creation ensures this ordering.

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
  "last_poll_at": "2026-04-23T10:30:05Z",
  "cache_age_seconds": 3,
  "kashif_reachable": true
}
```

**Response (503)** -- Janissary is not ready:

```json
{
  "status": "initializing",
  "has_state": false,
  "last_poll_at": null,
  "cache_age_seconds": null,
  "kashif_reachable": false
}
```

The healthcheck returns 503 until the first successful Divan poll AND
Kashif health check. This prevents Docker from routing dependent
services (Aga, Vizier) until the full security perimeter is warm.

### Graceful shutdown

On `SIGTERM` (Docker stop):

1. mitmproxy stops accepting new connections
2. In-flight requests complete (mitmproxy default: 5s drain)
3. DivanPoller thread stops (`_running = False`)
4. Appeal API server shuts down (uvicorn handles SIGTERM)
5. WireGuard interface torn down (`wg-quick down`)
6. Process exits

No state persistence needed -- all state is in Divan. On restart,
Janissary re-polls and rebuilds the cache.

### Failure modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| WireGuard setup fails at start | Janissary container exits, no traffic flows | Docker `restart: unless-stopped` retries. Fix WireGuard config. |
| Divan down at Janissary start | Janissary blocks all traffic (fail-closed) | Auto-recovers when Divan comes up (poller retries every 5s) |
| Divan goes down after start | Janissary uses cached state | Auto-recovers on next successful poll |
| Kashif down | Appeals auto-escalate to Sultan (never auto-approved) | Auto-recovers when Kashif comes up |
| OpenBao sealed | Aga cannot mint new tokens. Existing grants with valid leases still work; expired leases fail closed as usual. | Sultan unseals OpenBao |
| Janissary crashes | All province/Vizier traffic fails (WireGuard tunnel drops, kill-switch blocks all) | Docker `restart: unless-stopped` restarts it. Sultan has SSH fallback. |
| wg-client sidecar crashes | That province's traffic is killed (iptables kill-switch) | Docker restarts sidecar. Province traffic resumes when tunnel re-establishes. |
| Appeal API crashes | Appeals fail, proxy still works | Restart container. Traffic rules unaffected. |
| mitmproxy crashes | All traffic blocked (proxy gone, kill-switch active) | Docker restart. Fail-closed by design. |
