# Sultanate -- Secure Agent Deployment Platform

## The Problem

Container orchestration, network policies, and credential management are
solved problems -- for microservices operated by DevOps teams. AI agents are
a different workload with different constraints:

1. **Different trust model** -- a microservice does what its code says; you
   audit it by reading source. An LLM agent is non-deterministic. You can't
   predict its behavior from its configuration. Every agent must be treated
   as potentially confused or adversarial. This demands an assume-adversarial
   security posture, not assume-correct.

2. **Different operator profile** -- Kubernetes assumes a DevOps team writing
   YAML manifests. An individual running personal AI agents doesn't want to
   hand-craft networking, credential distribution, and monitoring for each
   one. Management should be conversational, not declarative.

3. **Different lifecycle** -- microservices are deploy-and-forget,
   request-response. Agent workloads are interactive, long-lived sessions
   with a human in the loop. The operator steers agents through conversation,
   not CI/CD pipelines.

Existing tools weren't designed for these constraints. **Sultanate is
container orchestration re-shaped for AI agents:** Vizier handles deployment,
Janissary handles security, Kashif inspects content, Divan holds the shared
state, and Sultan steers the whole operation through natural conversation.

## What is Sultanate

**Sultanate is your personal AI staff.** A system that lets you run multiple
AI agents -- each handling a distinct agenda -- with frictionless deployment
and enterprise-grade security.

## Who is this for?

Sultanate is built for a technical operator who wants to run multiple AI
agents as a personal workforce. The operator is comfortable with CLI and
Docker but doesn't want to hand-craft networking, credential distribution,
and monitoring for every agent.

Phase 1 assumes one operator (Sultan), one host machine, and conversational
management via Telegram.

## Architecture

Each agent runs in an isolated province (container). A province can hold:
- A single-purpose agent -- a personal assistant, a lawyer, a project tracker,
  a Wikipedia editor
- An agentic coding tool -- OpenHands, SWE-agent, Aider
- A multi-agent runtime -- CrewAI, AutoGen, LangGraph
- Any other workload that runs in Docker

From Sultanate's perspective, all of these are just "a container that needs
deployment and security."

**Four roles, clear boundaries:**

- **Sultan** (you) -- steers agents through conversation, approves access
  requests, reviews output. Communicates via whatever channel the agent
  runtime provides.
- **Vizier** (deployment orchestrator) -- your chief of staff. Creates
  provinces (isolated containers) from firmans (container templates) and
  berats (agent profiles), manages the roster, tracks what's running. Has
  a CLI for direct management.
- **Janissary** (egress proxy) -- dumb, deterministic security gate.
  Forwards province traffic through whitelist/blacklist/outbound-size-gate
  rules. No LLM, no content evaluation. Credential injection via grant table.
- **Kashif** (content inspector) -- paranoid local LLM that screens all
  content for malice. Handles Layer 4 appeal triage and screens all
  Aga ingress. Single question: "can this be in any way malicious?"
  Fail-closed: if down or unsure, block and alert Sultan.
- **Aga** (security advisor) -- trusted, operator-facing agent that
  manages secrets, contextualizes alerts, curates blacklists, and reviews
  access requests. All inputs pre-screened by Kashif.
- **Divan** (shared state store) -- all components read from and write to
  Divan. Includes a web dashboard for Sultan. Not an orchestrator -- just
  a registry and API.

**Independently deployable.** Vizier and the security perimeter
(Janissary + Kashif + Aga) are separate products in separate repos.
They compose together but can be deployed independently. Firmans and berats
may also live in their own repos.

**Hermes-native, Phase 1.** Aga and Vizier are implemented as Hermes
agents. The infrastructure layer (Janissary, Kashif, Divan) is
runtime-independent. Phase 2 adds OpenClaw support for Aga and Vizier.
Phase 3 targets runtime-agnostic berats that work across multiple runtimes.

## The Ottoman Metaphor

The naming isn't decorative -- it maps to a governance model:

| Name | Role | Analogy |
|------|------|---------|
| **Sultan** | Human operator | The ruler -- decides, approves, overrides |
| **Sultanate** | The whole system | The empire -- everything under Sultan's rule |
| **Vizier** | Deployment and orchestration | Grand Vizier -- manages the court, executes Sultan's will |
| **Janissary** | Egress proxy (dumb, no LLM) | Elite guard corps -- protects the gates, follows rules |
| **Kashif** | Content inspector (local LLM) | The taster -- inspects everything for poison before it reaches the court |
| **Aga** | Chief of security (trusted agent) | Agha of the Janissaries -- commands the guard corps, directs Kashif's inspections, manages secrets, reports to Sultan |
| **Divan** | Shared state store | The imperial council registry -- records everything, decides nothing |
| **Province** | Isolated container | A governed territory -- has its own governor, boundaries, and rules |
| **Pasha** | Agent inside a province | Provincial governor -- runs the territory, reports up |
| **Firman** | Container template | Imperial decree -- defines the office (Docker image, bootstrap, runtime) |
| **Berat** | Agent profile | Letter of appointment -- defines the governor (soul, tools, permissions) |
| **Realm** | All active provinces | The empire's territories -- what Sultan surveys |

The metaphor works because it encodes the trust hierarchy: Sultan trusts
Vizier, Janissary, Kashif, and Aga (they run with elevated privileges),
but trusts no Pasha (agent inside a province) fully -- every province is
isolated and monitored. Aga is trusted but guarded: all its inputs are
screened by Kashif before ingestion. This matches the security model
directly.

## Deployment Model

**Single host machine, Phase 1.** Everything runs on one server with Unix
user-level isolation:

| Component | User | Permissions | Why |
|-----------|------|-------------|-----|
| **Janissary** | root | Network control, iptables | Needs networking to enforce egress as proxy |
| **Kashif** | root (runs alongside Janissary) | Local LLM access, Divan read | Screens content for malice, no secrets access, no outbound of its own |
| **Aga** | root | Full host access, secret management | Manages secrets, reads audit state, contextualizes alerts |
| **Vizier** | dedicated user (`vizier`) | Docker group, no root | Creates/manages containers, writes province state to Divan |
| **Divan** | runs on host | Accessible to all components | Shared state store, no elevated permissions needed |
| **Provinces** | containerized | No host access, no direct internet | All egress through Janissary. Isolated workspace per province |

**Network topology:**

```text
Province A --+
Province B --+-- HTTP_PROXY --> Janissary --> Internet (whitelisted only)
Province C --+                      |
                                    +-- Kashif (screens appeals + Aga ingress)
                                    +-- Divan (shared state)
                                    +-- Secret Vault (OpenBao, local)
                                    +-- Aga --> Sultan (alerts with context)
```

- Provinces sit on an internal Docker network (`internal: true`, no external
  route)
- Janissary is the only component bridging internal and external networks
- Janissary itself has no outbound access -- it only forwards traffic and
  talks to Divan and OpenBao (both local). A compromised Janissary cannot
  become an open relay.
- Kashif screens all appeals and all Aga ingress (Pasha-originated
  content, fetched web pages) before delivery. Fail-closed: if Kashif is
  down or unsure, block and alert Sultan.
- Aga's outbound goes through Janissary with whitelist-only policy.
  Aga cannot expand its own whitelist -- only Sultan can. Any web
  content Aga fetches is inspected by Kashif before ingestion.
- Every alert passes through Aga first, which adds operator-facing
  context before reaching Sultan
- Only HTTP/HTTPS traffic supported (Phase 1). Non-HTTP protocols blocked
  by network topology.

**One-way dependencies via Divan:**

```text
Vizier ---writes---> Divan <---reads--- Janissary
                       ^
                       |
              reads/writes
                       |
                   Aga
                       ^
                       |
                  screened by
                       |
                    Kashif ---reads---> Divan (audit context)
```

No component calls another directly. All coordination happens through Divan.
Kashif screens all Pasha-originated content before it reaches Aga.

**Repo structure:**

| Repo | Contents |
|------|----------|
| `sultanate` | Superproject -- umbrella docs, deployment guide, submodules |
| `vizier` | Orchestration, province lifecycle, CLI |
| `janissary` | Egress proxy (Janissary), content inspector (Kashif), security advisor (Aga), Divan, audit |
| `hermes-firman` | Hermes container template (Docker image, bootstrap, runtime startup) |
| `hermes-coding-berat` | Coding agent profile (soul, tools, security policy) |

All component repos are git submodules of `sultanate`. Additional firmans
and berats get their own repos as needed.

## Component Overview

**Vizier** -- reactive realm (fleet of provinces) manager. Creates provinces
from firmans (container templates) and berats (agent profiles), launches
agents, tracks province lifecycle (creating -> running -> stopped ->
destroying). Writes province state to Divan. Provides a CLI for direct
management. Does not invent work -- acts on Sultan's commands. See
`VIZIER_PRD_V3.md`.

**Janissary** -- dumb, deterministic egress proxy. Reads rules from Divan and
applies them by source IP. Whitelist pass, size gate block (outbound request
payloads only), blacklist block. Blocked requests with appeals are routed to
Kashif. Transparent credential injection via grant table. No LLM, no content
evaluation, no outbound access of its own. See `JANISSARY_PRD_V2.md`.

**Kashif** -- paranoid content inspector running a local LLM on the host.
Screens all Pasha-originated content before it reaches Aga (appeal
justifications, access requests, freeform input). Screens fetched web content
before Aga ingests it. Handles Layer 4 appeal triage (approve obvious
safe, block obvious bad, escalate unclear to Aga). Single question: "can
this be in any way malicious?" Fail-closed: if down or unsure, block and alert
Sultan. Ships with Janissary.

**Aga** -- trusted, operator-facing Hermes agent running as root. Manages
secrets (creation, rotation, revocation), contextualizes all alerts before
they reach Sultan, curates the blacklist, reviews access requests, answers
audit queries. Ships with Janissary. Not part of deterministic enforcement.
All Aga inputs are pre-screened by Kashif for prompt injection and
manipulation. Aga cannot expand its own whitelist -- only Sultan can.

**Divan** -- shared state store and API. Holds province registry, grant table,
whitelists, blacklist, and audit log. All components communicate through
Divan, not directly. Includes a read-only web dashboard for Sultan. Ships
with Janissary.

**Province** -- the isolation unit. A container with its own workspace, agent,
privileges, and outbound policy. Can contain a single agent or an entire
multi-agent runtime. No direct internet access.

**Firman** -- a reusable container template. Defines the province's
infrastructure: Docker image, workspace bootstrap, runtime startup. A firman
is the office -- it says nothing about who works there. Phase 1 ships one:
`hermes-firman`.

**Berat** -- a reusable agent profile. Defines the Pasha's identity: soul
(personality), operating instructions, tool selection, and security policy
(whitelist, grants). A berat is the letter of appointment -- it says who the
governor is and what they can access. A province is created from a firman +
a berat.

**Pasha** -- the agent running inside a province. For Hermes provinces, this is
a Hermes agent. For other runtimes, it's whatever the runtime provides. Sultan
can communicate with a Pasha directly.

## Failure Modes

All components fail closed. If Divan is unreachable, Janissary enforces
last-cached rules (whitelist, blacklist, grants). If Janissary has never
successfully read from Divan (fresh start, no cache), it blocks all traffic.
Kashif blocks all content if its LLM is unresponsive or times out. Aga
alerts Sultan if it cannot reach Divan. No component fails open.

## Communication Model

**Phase 1: one Telegram bot per agent.** Sultan communicates with each Pasha,
Vizier, and Aga through separate Telegram bots in dedicated threads.
Each agent has its own bot token (provisioned by Aga). Communication is
1:1 -- Sultan to agent, agent to Sultan.

**Phase 2: shared Telegram channels.** Multiple agents and Sultan in the same
channel for cross-agent coordination, shared visibility, and group discussion.
Requires routing logic (who responds to what) and message attribution.

## TODO: Easy Deployment

Phase 1 must ship with a single-command deployment experience on a fresh
Ubuntu server. Target:

```bash
# Clone and deploy the entire Sultanate stack
git clone --recursive https://github.com/stranma/sultanate.git
cd sultanate
./deploy.sh
```

**What `deploy.sh` must handle:**
- Install dependencies (Docker, Docker Compose)
- Pull/build all component images (Janissary, Kashif, Aga, Divan, Vizier)
- Create internal Docker network (provinces, no external route)
- Start Janissary (egress proxy) + Kashif (content inspector) + Aga
  (security advisor) + Divan (state store)
- Start Vizier (orchestrator)
- Prompt Sultan for initial configuration:
  - Telegram bot tokens (or auto-create)
  - OpenBao initialization and unseal (Sultan holds unseal key(s))
  - Sultan's Telegram user ID
- Validate connectivity (Janissary reachable, Divan healthy, Aga online)

**What creating a province should look like:**
```bash
vizier create hermes-firman --berat hermes-coding-berat \
  --repo stranma/EFM --name backend-refactor
```

Or via Telegram to Vizier: "Create a coding province for stranma/EFM."

**Per-component deployment:**
Each component repo must also be independently deployable for development
and testing:
- `janissary/`: `docker compose up` starts Janissary + Kashif + Aga + Divan
- `vizier/`: `docker compose up` starts Vizier (requires Divan endpoint)
- `hermes-firman/`: `docker build` produces the province base image

## Phase 1 Scope

Single host deployment with Unix user-level isolation.

**Vizier:** province lifecycle (create, start, stop, destroy), CLI, firman +
berat based province creation. One firman (`hermes-firman`), one berat
(`hermes-coding-berat`).

**Janissary:** HTTP/HTTPS egress proxy with CONNECT tunnel support, 3-layer
traffic model (whitelist, size gate on outbound payloads, blacklist),
transparent credential injection via grant table, security MCP tool. No LLM,
no content evaluation, no outbound access of its own.

**Kashif:** local LLM content inspector. Layer 4 appeal triage, Aga
ingress screening, fetched content inspection. Fail-closed. Ships with
Janissary.

**Aga:** secret management (create, rotate, revoke), alert
contextualization, blacklist curation, access request review. All inputs
screened by Kashif. Ships with Janissary, non-optional.

**Divan:** SQLite + HTTP API. Province registry, grant table, whitelists,
blacklist, audit log, read-only web dashboard.

**Runtime:** Hermes-native. Aga and Vizier are Hermes agents.
Infrastructure layer (Janissary, Kashif, Divan) is runtime-independent.

**Deferred:**
- Shared Telegram channels for multi-agent coordination (Phase 2)
- OpenClaw support for Aga and Vizier (Phase 2)
- Firman/berat boundary review -- security policy ownership, tool/firman
  compatibility validation (Phase 2)
- Additional firmans and berats (Phase 2)
- Cross-province coordination (Phase 2)
- Multi-machine deployment (Phase 3)
- Task tracking in Divan -- task state distinct from province state (Phase 3)
- Runtime-agnostic berats (e.g., `coding-berat` that works across Hermes,
  OpenClaw, and other runtimes) (Phase 3)
- Cost and budget reporting (Phase 3)
- Multi-Sultan support (Phase 3)
