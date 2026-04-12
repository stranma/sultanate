# Sultanate

Secure agent deployment platform -- deploy AI agents fast via conversation,
keep secrets safe, and swap in new runtimes easily.

Start with [MOTIVATION.md](MOTIVATION.md) for why, then
[SULTANATE_MVP.md](SULTANATE_MVP.md) for the architecture.

## Status: Spec Phase

MVP PRDs and technical specs are complete. Implementation next.

## Documents

### Motivation & Architecture

| Document | Description |
|----------|-------------|
| [MOTIVATION.md](MOTIVATION.md) | Problem, insight, vision |
| [SULTANATE_MVP.md](SULTANATE_MVP.md) | MVP architecture, trust model, components |

### MVP PRDs (what to build)

| Document | Component |
|----------|-----------|
| [JANISSARY_MVP_PRD.md](JANISSARY_MVP_PRD.md) | Egress proxy + credential injection |
| [SENTINEL_MVP_PRD.md](SENTINEL_MVP_PRD.md) | Secret management agent |
| [VIZIER_MVP_PRD.md](VIZIER_MVP_PRD.md) | Province orchestration CLI + agent |
| [HERMES_FIRMAN_MVP_PRD.md](HERMES_FIRMAN_MVP_PRD.md) | Hermes container template |
| [HERMES_CODING_BERAT_MVP_PRD.md](HERMES_CODING_BERAT_MVP_PRD.md) | Coding agent profile |

### Technical Specs (how to build)

| Document | Component |
|----------|-----------|
| [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md) | Shared state API contract (all components depend on this) |
| [JANISSARY_SPEC.md](JANISSARY_SPEC.md) | Sandcat/mitmproxy proxy, CA cert, appeal API |
| [SENTINEL_SPEC.md](SENTINEL_SPEC.md) | Infisical integration, grant lifecycle, iptables |
| [VIZIER_SPEC.md](VIZIER_SPEC.md) | CLI, Docker integration, berat/firman loading |
| [HERMES_FIRMAN_SPEC.md](HERMES_FIRMAN_SPEC.md) | firman.yaml schema, upstream image usage |
| [HERMES_CODING_BERAT_SPEC.md](HERMES_CODING_BERAT_SPEC.md) | berat.yaml schema, templates, security policy |

### Full PRDs (future scope, reference only)

| Document | Description |
|----------|-------------|
| [SULTANATE.md](SULTANATE.md) | Full architecture with Kashif, Divan dashboard, SentinelGate |
| [JANISSARY_PRD_V2.md](JANISSARY_PRD_V2.md) | Full security perimeter (4-layer traffic, Kashif, Sentinel) |
| [VIZIER_PRD_V3.md](VIZIER_PRD_V3.md) | Full orchestrator |
| [HERMES_FIRMAN_PRD_V1.md](HERMES_FIRMAN_PRD_V1.md) | Full container template |
| [HERMES_CODING_BERAT_PRD_V1.md](HERMES_CODING_BERAT_PRD_V1.md) | Full coding agent profile |
| [SENTINELGATE_ANALYSIS.md](SENTINELGATE_ANALYSIS.md) | SentinelGate integration analysis |

## Repo Structure

| Repo | Contents |
|------|----------|
| `sultanate` (this repo) | Umbrella docs, specs, deployment guide |
| [`vizier`](https://github.com/stranma/Vizier) | Province orchestration CLI + Hermes agent |
| [`janissary`](https://github.com/stranma/janissary) | Egress proxy (Janissary), shared state (Divan), secret management (Sentinel) |
| [`hermes-firman`](https://github.com/stranma/hermes-firman) | Hermes container template |
| [`hermes-coding-berat`](https://github.com/stranma/hermes-coding-berat) | Coding agent profile |
