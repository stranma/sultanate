# PRD: Vizier v3 -- Province Orchestration for Sultanate

> For shared glossary, deployment model, and component overview see
> [SULTANATE.md](SULTANATE.md).

## Vision

Vizier is the deployment and orchestration layer for Sultanate. It creates
provinces (isolated containers) from firmans (container templates), launches
agents, manages the realm (fleet of all provinces), and writes province state
to Divan (shared state store). It has a CLI for direct management and
communicates with Sultan (human operator) via whatever channel the agent
runtime provides.

Vizier is reactive -- it acts on Sultan's commands and agent events. It does
not invent work on its own.

Vizier does not handle security (Janissary's job), content inspection
(Kashif's job), or secret management (Aga's job). It writes province
state to Divan and trusts the security perimeter to enforce policy.

Phase 1 is Hermes-native. Phase 2 adds OpenClaw support.

## Product Boundary

**Vizier provides:**
- Province lifecycle management (create, start, stop, destroy)
- Firman-based province creation (templates)
- Realm tracking (what's running, what state it's in)
- CLI for direct management
- Province state reporting to Divan

**Vizier does NOT provide:**
- Network security or egress control (Janissary's job)
- Content inspection (Kashif's job)
- Secret management or credential injection (Aga's job)
- Agent runtime (Hermes or other runtime's job)
- Task decomposition or work planning (Pasha's job)

## Design Constraints

- **Vizier is reactive.** It acts on Sultan's commands and province events.
  It does not invent work, schedule tasks, or decide what agents should do.
- **Vizier is not root.** It runs as a dedicated user (`vizier`) with Docker
  group access. It can create and manage containers but cannot modify network
  rules, access secrets, or read audit state directly.
- **Vizier does not call Janissary, Kashif, or Aga.** All coordination
  happens through Divan. Vizier writes province state; Aga reads it and
  provisions security.
- **Provinces are long-lived.** A province may handle multiple tasks over
  time. Province lifecycle state is not task state.
- **Hermes-native, Phase 1.** Vizier creates containers. What runs inside
  is determined by the firman. Phase 1 uses Hermes. Phase 2 adds OpenClaw.

## Province Lifecycle

Province state is infrastructure state, not task state:

- **creating** -- Vizier is instantiating the province from a firman, bringing
  up the container and configuring the runtime environment
- **running** -- the province exists, the agent is reachable, workspace and
  proxy configuration are active
- **stopped** -- the province exists but is not currently running
- **failed** -- province startup or runtime has failed, operator attention
  required
- **destroying** -- Vizier is tearing down the province and cleaning up

Vizier writes every state change to Divan. Aga watches Divan for new
provinces and provisions security (grants, whitelist) accordingly.

## Province Creation Flow

```text
1. Sultan tells Vizier: "Create a province for X"
2. Vizier selects (or Sultan specifies) a firman
3. Vizier creates the container:
   --> internal Docker network only (no external route)
   --> HTTP_PROXY / HTTPS_PROXY pointing to Janissary
   --> workspace bootstrapped per firman spec
   --> agent runtime started per firman spec
4. Vizier writes to Divan:
   --> province ID, container IP, status=creating, firman used
5. Aga reads new province from Divan:
   --> provisions default grants from firman
   --> sets up whitelist from firman defaults
6. Vizier updates Divan: status=running
7. Sultan can now communicate with the Pasha inside
```

## Firmans and Berats

A province is created from a **firman** (container template) and a **berat**
(agent profile). See [SULTANATE.md](SULTANATE.md) for definitions.

**Firman** (container template) -- the office. Defines infrastructure:

- **Container image** -- what Docker image to use
- **Workspace bootstrap** -- repo cloning, directory structure
- **Runtime startup** -- how to start the agent runtime inside the container

**Berat** (agent profile) -- the employee. Defines the agent:

- **Soul** -- Pasha personality and operating style
- **Instructions template** -- operating rules, role definition
- **Tool selection** -- what tools are available to the Pasha
- **Security policy** -- initial whitelist, default grants, size gate threshold

Vizier is firman-agnostic and berat-agnostic. It instantiates provinces from
the combination and tracks their lifecycle.

Phase 1 requires one firman (`hermes-firman`) and one berat
(`hermes-coding-berat`).

## Realm Management

The realm is the set of all provinces. Vizier tracks realm state in Divan
and reports it to Sultan on request.

**Sultan can ask:**
- "What is running right now?" -- Vizier lists active provinces with status
- "Stop province X" -- Vizier stops the province, updates Divan
- "Kill province X" -- Vizier destroys the province, updates Divan (Aga
  revokes grants on seeing the state change)
- "Restart province X" -- Vizier restarts a stopped province

## Sultan-Pasha Direct Communication

Vizier owns province creation and realm coordination, but Sultan may message
a Pasha (agent inside a province) directly for:
- Execution decisions ("focus on the API first")
- Clarification ("what's the tradeoff here?")
- Status ("where are you on this?")

This happens through the agent runtime's channel (e.g., Telegram thread for
Hermes). Vizier is not in this communication path -- it's direct.

## CLI

Vizier provides a CLI for direct management:

```
vizier create <firman> [--name <name>]   # create province from firman
vizier list                               # list all provinces with status
vizier status <province>                  # detailed province status
vizier stop <province>                    # stop a running province
vizier start <province>                   # start a stopped province
vizier destroy <province>                 # destroy province and clean up
vizier logs <province>                    # view province logs
```

The CLI writes to Divan the same way conversational commands do. It is an
alternative interface, not a separate system.

## Phase 1 Scope

**In scope:**
- Province lifecycle management (create, start, stop, destroy)
- Province state reporting to Divan
- Firman-based province creation
- One firman: `hermes-firman` (Hermes agent, GitHub PR delivery)
- Container creation with internal-only Docker network
- HTTP_PROXY/HTTPS_PROXY configuration pointing to Janissary
- Realm status reporting to Sultan
- CLI for direct management
- Sultan-Pasha direct communication via runtime channel

**Deferred:**
- OpenClaw support (Phase 2)
- Cross-province coordination and shared channels (Phase 2)
- Multiple firmans beyond `hermes-firman` (Phase 2)
- Multi-machine deployment (provinces across hosts) (Phase 3)
- Cost and budget reporting (Phase 3)
- Province resource limits and quotas (Phase 3)
- Automatic province scaling (Phase 3)
