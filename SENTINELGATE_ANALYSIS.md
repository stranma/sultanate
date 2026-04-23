# SentinelGate + Sultanate: Integration Analysis

## Context

This analysis maps [SentinelGate](https://github.com/Sentinel-Gate/Sentinelgate)
capabilities to Sultanate's security architecture (Janissary + Kashif + Aga)
to determine what SentinelGate could replace, complement, or accelerate.

## Architecture Comparison

### What SentinelGate is

An MCP proxy (Go, single binary) that sits between AI agents and their tool
servers. Every tool call passes through an interceptor chain:

```
Agent Request
  -> Validation (JSON-RPC, confused deputy protection)
  -> Authentication (API key -> Identity with roles)
  -> Audit (entry point logging)
  -> Rate Limiting (per-IP, per-user)
  -> Quotas (per-identity call/write/delete limits)
  -> Policy Evaluation (CEL expressions)
  -> Approval (human-in-the-loop if needed)
  -> Content Scanning (PII, secrets, prompt injection in args)
  -> Response Scanning (prompt injection in upstream responses)
  -> Upstream Router (dispatch to correct MCP server)
  -> Response Transforms (redact, mask, truncate, inject warnings)
  -> Audit (log decision + response)
```

### What Sultanate's security layer is

A network-level egress proxy (Janissary) + content inspector (Kashif) +
security advisor agent (Aga) + shared state store (Divan):

```
Province Container
  -> HTTP_PROXY -> Janissary (network-level, dumb, deterministic)
     -> Layer 1: Whitelist (pass silently)
     -> Layer 2: Size gate (block large outbound payloads)
     -> Layer 3: Blacklist (block known bad)
     -> Layer 4: Appeal -> Kashif (LLM triage) -> Aga -> Sultan
  -> Credential injection at proxy (grant table from Divan)
  -> Audit trail to Divan
```

### Key Architectural Difference

| Dimension | SentinelGate | Janissary |
|-----------|-------------|-----------|
| **Enforcement layer** | Application (MCP protocol) | Network (HTTP proxy) |
| **Bypassability** | Agent must use the MCP proxy | Agent cannot bypass (it's the network route) |
| **Protocol awareness** | Understands tool names, schemas, arguments | Sees HTTP requests (URL, headers, payload size) |
| **Credential handling** | Scans for leaked secrets in args | Injects secrets transparently (agent never sees them) |
| **Content inspection** | Regex-based PII/secret patterns | LLM-based paranoid triage (Kashif) |
| **Intelligence layer** | CEL policies + session tracking | Aga agent (LLM, operator-facing) |
| **State store** | In-memory + state.json file | Divan (SQLite + HTTP API, shared across all components) |
| **Scope** | Tool calls only | All outbound HTTP/HTTPS traffic |

**Critical insight:** These operate at different layers and are complementary,
not competing. SentinelGate controls what tools an agent can call and with what
arguments. Janissary controls what network traffic leaves the container. An agent
could have tools that make HTTP requests -- SentinelGate governs the tool call,
Janissary governs the actual network request.

## Component Mapping

### What SentinelGate could replace or accelerate

#### 1. Kashif (Content Inspector) -- PARTIAL REPLACEMENT

Kashif's job: screen content for malice, triage appeals, detect prompt injection.

SentinelGate provides:
- **Content scanning** (`internal/domain/action/content_scanner.go`): regex-based
  detection of PII (emails, SSN, CC numbers), secrets (AWS keys, GitHub tokens,
  API keys), with configurable actions (block, mask, alert)
- **Response scanning** (`internal/domain/action/response_scanner.go`): prompt
  injection detection in upstream responses
- **Whitelisting**: per-field, per-scope bypass for legitimate use cases

What's missing for Kashif:
- **LLM-based triage**: Kashif uses a local LLM to answer "can this be malicious?"
  SentinelGate uses regex patterns only. Regex catches known patterns; LLM catches
  novel or contextual attacks.
- **Fail-closed paranoid mode**: Kashif blocks when unsure. SentinelGate's regex
  either matches or doesn't -- no "unsure" state.
- **Appeal workflow for blocked traffic**: Kashif triages Layer 4 appeals (approve
  obvious safe, block obvious bad, escalate unclear). SentinelGate has an approval
  interceptor but it's for policy-triggered holds, not content-triggered appeals.

**Verdict:** SentinelGate's content scanning is a fast first pass (Layer 0 before
Kashif). It catches known patterns deterministically. Kashif remains needed for
contextual/novel attack detection and the appeal triage workflow.

#### 2. Policy Engine -- STRONG FIT

Janissary's policy model is simple: whitelist/blacklist/size-gate tables read
from Divan. SentinelGate's CEL engine is far more expressive:

```cel
// Block deletion tools for non-admins
tool_name.startsWith("delete_") && !("admin" in user_roles)

// Detect exfiltration pattern: many writes in short window
session_write_count > 10

// Block uploads to paste sites
dest_domain_matches(dest_domain, "*.pastebin.com")

// Session-aware: read sensitive file then upload = suspicious
"read_file" in session_action_set && action_type == "http_request"
```

This maps to Sultanate as follows:
- **Whitelist/blacklist** -> CEL rules with `dest_domain_matches()`
- **Size gate** -> CEL rule on payload size (would need custom variable)
- **Per-province policies** -> Identity-scoped policies (province = identity)
- **Grant table logic** -> Could express as CEL rules per identity + destination

**Verdict:** SentinelGate's CEL engine could power the policy layer for both
Janissary (network rules expressed as CEL) and the MCP-level tool policies.
Janissary stays dumb -- it reads compiled rules, not CEL directly -- but the
rule authoring and evaluation could use SentinelGate's engine.

#### 3. Audit System -- COMPLEMENTARY

Sultanate: audit log in Divan (SQLite), queryable by Aga, shown on dashboard.

SentinelGate: ECDSA P-256 signed audit records with per-record cryptographic
signatures. Records include tool name, arguments (redacted), decision, latency,
scan detections, transform results.

**Verdict:** SentinelGate's signed audit records are strictly better for
tamper-proof evidence. Sultanate should adopt this for Divan's audit log.
The signing approach (`internal/adapter/outbound/evidence/signer.go`) is
clean and portable.

#### 4. Session Tracking -- NEW CAPABILITY

Sultanate has no session-level behavioral tracking. Janissary sees individual
HTTP requests with no cross-request correlation.

SentinelGate tracks per-session:
- Call counts (total, read, write, delete)
- Action history (last N tool calls, FIFO)
- Unique tool names used (action set)
- Unique argument keys (arg key set)
- Cumulative cost
- Duration and idle detection

This enables exfiltration detection policies that Sultanate currently cannot
express. Example: "province read 50 files then tried to POST to an external
URL" requires cross-request correlation.

**Verdict:** Adopt session tracking for province-level behavioral analysis.
Feed session metrics into Aga's alert contextualization.

#### 5. RBAC and Identity -- REUSABLE

Sultanate identifies provinces by source IP on the Docker network. SentinelGate
uses API keys mapped to identities with roles.

For MCP-level control (inside the province), SentinelGate's identity model maps
naturally: each province gets an API key, identity = province ID, roles derived
from berat security policy.

**Verdict:** Use SentinelGate's identity model for MCP-level access control
within provinces. Janissary continues to use source IP for network-level
identification (it doesn't speak MCP).

### What SentinelGate cannot replace

#### 1. Janissary (Network Egress Proxy) -- CANNOT REPLACE

SentinelGate operates at the MCP application layer. It has no network-level
enforcement. A compromised or misbehaving agent that makes raw HTTP requests
(curl, wget, Python requests) bypasses SentinelGate entirely.

Janissary is the network route -- there is no bypass. This is a fundamental
security property that application-level controls cannot provide.

**Janissary remains essential.** Sandcat (mitmproxy + WireGuard) is still the
right foundation for the network layer.

#### 2. Aga (Security Advisor Agent) -- CANNOT REPLACE

Aga is an LLM agent that:
- Manages secrets (create, rotate, revoke via OpenBao)
- Contextualizes alerts for Sultan ("Province B tried to push to a new repo,
  here's why this might be legitimate or suspicious")
- Curates blacklists based on observed patterns
- Reviews escalations with broader operational context
- Answers Sultan's audit queries in natural language

SentinelGate has no LLM, no secret management, no operator-facing intelligence.
Its policies are static rules, not contextual judgments.

**Aga remains essential.**

#### 3. Divan (Shared State) -- CANNOT REPLACE

SentinelGate uses in-memory stores with optional file persistence. Sultanate
needs a shared state store accessible to Vizier, Janissary, Aga, Kashif,
and the web dashboard. Different access patterns, different writers.

**Divan remains essential.**

#### 4. Credential Injection -- DIFFERENT MODEL

Janissary + Sandcat: proxy-level injection. Agent sends request with placeholder,
proxy substitutes real credential before forwarding. Agent never sees the secret.

SentinelGate: scans for accidentally leaked secrets in tool arguments. It does
not inject credentials -- it prevents their exposure.

These are complementary: Janissary injects, SentinelGate detects leaks.

## Proposed Integration Architecture

```
Province Container
  |
  +-- Pasha (agent) uses MCP tools
  |     |
  |     +-- SentinelGate (MCP proxy, runs inside or alongside province)
  |           -> Tool-level RBAC (berat-derived policies)
  |           -> CEL policy evaluation
  |           -> Content scanning (fast regex, first pass)
  |           -> Session tracking (behavioral analysis)
  |           -> Signed audit records
  |           -> Response transforms (redact secrets in tool outputs)
  |           |
  |           +-- Upstream MCP servers (filesystem, git, etc.)
  |
  +-- All HTTP traffic (including MCP upstream calls that hit the network)
        |
        +-- Janissary (network egress proxy, Sandcat-based)
              -> Whitelist/blacklist/size-gate (from Divan)
              -> Credential injection (grant table)
              -> Blocked? -> Kashif (LLM triage) -> Aga -> Sultan
              -> Audit to Divan
```

### Two-layer security model

| Layer | Component | Enforces | Bypass risk |
|-------|-----------|----------|-------------|
| **Application** | SentinelGate | Tool-level policies, content scanning, session behavior | Agent uses raw HTTP instead of MCP tools |
| **Network** | Janissary/Sandcat | Egress filtering, credential injection, payload size | None (it's the network route) |

The network layer catches everything the application layer misses. The
application layer provides richer context (tool names, argument semantics,
session history) that the network layer cannot see.

### Data flow between layers

1. **Vizier** creates province, writes to Divan (province ID, IP, berat)
2. **Aga** reads new province, provisions credentials to Divan grant table
3. **SentinelGate** starts with province's berat-derived policy (CEL rules
   generated from berat security policy)
4. **Pasha** calls MCP tools -> SentinelGate evaluates, scans, audits
5. **MCP tool makes HTTP request** -> Janissary evaluates at network level
6. **SentinelGate session metrics** feed into Divan -> Aga uses for
   alert contextualization
7. **SentinelGate detects anomaly** -> writes alert to Divan -> Aga
   contextualizes -> Sultan

### Berat-to-CEL compilation

A berat's security policy section could compile to SentinelGate CEL rules:

```yaml
# Berat security policy (Sultanate format)
security:
  tools:
    allow: ["read_file", "write_file", "list_directory", "bash"]
    deny: ["delete_*"]
  constraints:
    max_writes_per_session: 50
    max_file_size_kb: 100
```

Compiles to SentinelGate policy:

```yaml
policies:
  - name: "province-backend-refactor"
    rules:
      - name: "allow-specified-tools"
        condition: 'tool_name in ["read_file", "write_file", "list_directory", "bash"]'
        action: "allow"
      - name: "deny-delete"
        condition: 'tool_name.startsWith("delete_")'
        action: "deny"
      - name: "write-limit"
        condition: 'session_write_count > 50'
        action: "deny"
        help_text: "Session write limit exceeded (50). Request additional access."
      - name: "deny-unlisted"
        condition: "true"
        action: "deny"
```

This means Vizier can auto-generate SentinelGate configs from berats at
province creation time.

## Implementation Recommendations

### Phase 1: Adopt as library/sidecar

1. **Deploy SentinelGate as a sidecar** in each province container (or on
   the host proxying per-province). Single binary, zero dependencies.
2. **Generate SentinelGate config from berat** at province creation (Vizier).
   Berat security policy -> CEL rules + identity + API key.
3. **Feed SentinelGate audit to Divan** via a lightweight adapter (SentinelGate
   writes to file/stdout, a sidecar ships to Divan HTTP API).
4. **Keep Janissary/Sandcat as the network layer.** No change to egress proxy.
5. **Keep Kashif and Aga.** SentinelGate's regex scanning is Layer 0;
   Kashif's LLM screening is Layer 1 for appeals and escalations.

### Phase 2: Deeper integration

6. **Session metrics -> Aga context.** Aga reads SentinelGate's
   session data from Divan to enrich alerts ("Province A has made 200 tool
   calls in 5 minutes, 80% writes, targeting files matching `*.env`").
7. **Signed audit records in Divan.** Adopt SentinelGate's ECDSA signing
   for all Divan audit entries (not just MCP-level).
8. **CEL as the universal policy language.** Janissary's whitelist/blacklist
   tables compiled from CEL rules stored in Divan. Single policy authoring
   surface for Sultan.

### What NOT to adopt

- **SentinelGate's in-memory state** -- Divan is the state store, not
  SentinelGate's `state.json`
- **SentinelGate's identity/auth as the primary identity system** -- provinces
  are identified by Docker network IP at the Janissary layer; SentinelGate
  identity is complementary for MCP-level control
- **SentinelGate's admin UI as the Sultan dashboard** -- Divan's web dashboard
  is the single pane of glass; SentinelGate admin is per-instance

## Summary

| Sultanate Component | SentinelGate Role | Action |
|---------------------|-------------------|--------|
| **Janissary** (egress proxy) | Cannot replace (network vs app layer) | Keep Sandcat. SentinelGate is complementary layer above. |
| **Kashif** (content inspector) | Partial overlap (regex vs LLM) | Use SentinelGate as fast first-pass scanner. Keep Kashif for LLM triage. |
| **Aga** (security advisor) | No overlap (no LLM, no secret mgmt) | Keep. Feed SentinelGate session data into Aga's context. |
| **Divan** (shared state) | No overlap (in-memory vs shared store) | Keep. SentinelGate writes audit to Divan. |
| **Policy engine** | Strong fit (CEL >> whitelist tables) | Adopt CEL as universal policy language across both layers. |
| **Session tracking** | New capability | Adopt. Critical for exfiltration detection. |
| **Audit signing** | Better than current design | Adopt ECDSA signing for Divan audit records. |
| **Tool-level RBAC** | New capability | Adopt. Berat security policy compiles to SentinelGate CEL rules. |
| **Response transforms** | New capability | Adopt. Redact secrets from tool outputs before agent sees them. |

**Bottom line:** SentinelGate is the application-level security layer that
Sultanate is missing. Janissary/Sandcat is the network-level layer that
SentinelGate is missing. Together they provide defense in depth -- network
enforcement catches everything, MCP-level enforcement adds semantic richness.
