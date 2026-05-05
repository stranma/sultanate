# Sultanate

Secure agent deployment platform -- run multiple AI agents as your personal
staff with frictionless deployment and enterprise-grade security.

## Status: Design Phase

This repo currently contains **PRDs and technical specs only** -- no
implementation yet. The documents define the MVP architecture, components,
security model, and Phase 1 scope.

## Where to Start

Read in this order for a cold start:

1. [MOTIVATION.md](MOTIVATION.md) -- why this exists, what problem it solves
2. [ARCHITECTURE.md](ARCHITECTURE.md) -- system diagrams, flow diagrams, user stories
3. [SULTANATE_MVP.md](SULTANATE_MVP.md) -- MVP scope, trust model, startup order
4. Component PRDs in the table below

## Document Index

### Cross-cutting

| Document | Role |
|----------|------|
| [MOTIVATION.md](MOTIVATION.md) | Problem statement + operator goals |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System diagrams, flow diagrams, 10 user stories with test assertions |
| [SULTANATE_MVP.md](SULTANATE_MVP.md) | Umbrella: MVP scope, trust model, credential model, component table, startup order, Hetzner hardware target |
| [CLAUDE.md](CLAUDE.md) | Guidance for Claude Code sessions (architectural invariants, naming, editing conventions) |

### Security perimeter (Janissary, Kashif, Aga, Divan, OpenBao)

| Document | Role |
|----------|------|
| [JANISSARY_MVP_PRD.md](JANISSARY_MVP_PRD.md) | Egress proxy scope (Sandcat fork, WireGuard, 4 traffic rules, Kashif-integrated appeal API) |
| [JANISSARY_SPEC.md](JANISSARY_SPEC.md) | Full implementation spec (mitmproxy addon, DivanPoller, lease-aware credential injection, Kashif client, Docker topology) |
| [KASHIF_MVP_PRD.md](KASHIF_MVP_PRD.md) | Content inspector -- three-layer CPU-only screener (LLM Guard regex, Prompt Guard 2 22M, Llama Guard 3 1B Q4) |
| [AGA_MVP_PRD.md](AGA_MVP_PRD.md) | Security chief -- OpenClaw agent managing secrets via OpenBao, running as root |
| [AGA_SPEC.md](AGA_SPEC.md) | Full Aga implementation (OpenBao AppRole, GitHub App minting, grant lifecycle, 5 polling loops) |
| [DIVAN_MVP_PRD.md](DIVAN_MVP_PRD.md) | Shared state store + dashboard scope (SQLite + FastAPI + Jinja2/HTMX) |
| [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md) | Full HTTP API spec -- 7 resources, role-based auth, SQLite schema, dashboard routes |

### Orchestration

| Document | Role |
|----------|------|
| [VIZIER_MVP_PRD.md](VIZIER_MVP_PRD.md) | Province orchestrator scope (vizier-cli + OpenClaw agent wrapper) |
| [VIZIER_SPEC.md](VIZIER_SPEC.md) | Full implementation (Click CLI, firman/berat YAML, template rendering, WireGuard peer alloc, Kashif-aware appeal relay) |

### Province runtime (OpenClaw Phase 1)

| Document | Role |
|----------|------|
| [OPENCLAW_FIRMAN_MVP_PRD.md](OPENCLAW_FIRMAN_MVP_PRD.md) | Container template scope (openclaw/openclaw image, no custom Dockerfile) |
| [OPENCLAW_FIRMAN_SPEC.md](OPENCLAW_FIRMAN_SPEC.md) | firman.yaml schema, bootstrap sequence, CA cert install, upstream image details |
| [OPENCLAW_CODING_BERAT_MVP_PRD.md](OPENCLAW_CODING_BERAT_MVP_PRD.md) | Coding agent profile: soul, tools, whitelist, grants, port declarations |
| [OPENCLAW_CODING_BERAT_SPEC.md](OPENCLAW_CODING_BERAT_SPEC.md) | berat.yaml schema, template rendering, grant provisioning flow, MCP config |

### Exploratory / historical

| Document | Role |
|----------|------|
| [SENTINELGATE_ANALYSIS.md](SENTINELGATE_ANALYSIS.md) | Integration analysis for [SentinelGate](https://github.com/Sentinel-Gate/Sentinelgate) -- kept for Phase 2 reference (session tracking, tool-level RBAC, ECDSA-signed audit) |

The pre-MVP Hermes + Infisical + Sentinel baseline is preserved on the
`origin/archive-hermes-infisical` branch.

## Implementation Notes

**Janissary + Sandcat:** The [Sandcat](https://github.com/VirtusLab/sandcat)
project is the foundation for Janissary. Sandcat already provides the core
primitives: a transparent mitmproxy-based egress proxy with
allowlist/deny-list network rules, and a credential-injection pattern that
substitutes placeholders with real secrets at the proxy level (so
containers never see actual credentials). The WireGuard tunnelling approach
routes all container traffic through the proxy without per-tool
configuration. Janissary's additional scope -- per-source-IP rules, Kashif
content triage, OpenBao lease awareness, Divan integration, appeal HTTP
API -- is built on top of the Sandcat fork.

**Content inspector Kashif:** three-layer, CPU-only design. Layer 1 is
[LLM Guard](https://github.com/protectai/llm-guard) regex scanners (MIT).
Layer 2 is Meta's [Prompt Guard 2 22M](https://github.com/meta-llama/PurpleLlama)
classifier. Layer 3 is Llama Guard 3 1B (Q4 quantization) as a paranoid LLM
judge. The whole pipeline runs comfortably on a Ryzen 5 3600 (~2-4 GB RAM
resident, ~1-2 s per appeal). See `KASHIF_MVP_PRD.md`.

**Secret Vault -- OpenBao:** Sultanate uses
[OpenBao](https://openbao.org/) (Apache 2.0 fork of HashiCorp Vault) as
the Secret Vault. Deployed as a single binary in a local container,
OpenBao holds the Sultanate GitHub App private key (KV) plus any
Sultan-pasted fallback tokens. Aga mints per-province GitHub App
installation tokens on demand (dynamic mode, 1-hour TTL, auto-renewed
every ~15 min while province is running). Sultan never pastes tokens in
the common path -- only performs the one-time GitHub App install. Phase
1 uses manual unseal at boot.

**Agent runtime -- OpenClaw:** [OpenClaw](https://openclaw.ai) is the
agent runtime for Phase 1. It supplies the Docker image
(`openclaw/openclaw:vYYYY.M.D`), gateway daemon (`openclaw gateway
--port 18789`), workspace-root auto-loaded files (`SOUL.md`, `AGENTS.md`,
`IDENTITY.md`), MCP server support, and multi-provider model support
(Anthropic / OpenAI / OpenRouter / Ollama / LiteLLM via one config
field). Vizier, Aga, and each Pasha run as OpenClaw agents.

**Target host -- Hetzner AX41-NVMe:** Ryzen 5 3600 (6c/12t), 64 GB DDR4,
2x512 GB NVMe, no GPU. Core Sultanate components fit in ~6-10 GB RAM,
leaving ~50 GB for provinces (5-10 concurrent coding agents comfortable).
GPU upgrade is a Phase 2 decision if Kashif throughput or local-LLM
Pashas become a need.

## Planned Repo Structure

Once implementation begins, component repos become git submodules here:

| Repo | Contents |
|------|----------|
| `sultanate` (this repo) | Umbrella docs, deployment guide, submodules |
| `vizier` | Orchestration, province lifecycle, `vizier-cli`, appeal relay |
| `janissary` | Egress proxy (Janissary), content inspector (Kashif), security chief (Aga), shared state (Divan) |
| `openclaw-firman` | OpenClaw container template (firman.yaml) |
| `openclaw-coding-berat` | Coding agent profile (berat.yaml + SOUL.md / AGENTS.md / IDENTITY.md / openclaw.json templates) |

Additional firmans and berats (e.g., for OpenHands or CrewAI runtimes,
research-berat, assistant-berat) are Phase 2.
