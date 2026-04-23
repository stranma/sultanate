# Sultanate

Secure agent deployment platform -- run multiple AI agents as your personal
staff with frictionless deployment and enterprise-grade security.

## Status: Design Phase

This repo currently contains **PRDs only** -- no implementation yet. The
documents define the architecture, components, security model, and Phase 1
scope.

## What's here

| Document | Defines |
|----------|---------|
| [SULTANATE.md](SULTANATE.md) | Umbrella architecture, trust model, deployment topology, component overview |
| [JANISSARY_PRD_V2.md](JANISSARY_PRD_V2.md) | Security perimeter: egress proxy (Janissary), content inspector (Kashif), security advisor (Aga), shared state (Divan) |
| [VIZIER_PRD_V3.md](VIZIER_PRD_V3.md) | Deployment orchestrator: province lifecycle, CLI, firman/berat templating |
| [HERMES_FIRMAN_PRD_V1.md](HERMES_FIRMAN_PRD_V1.md) | Container template for Hermes-based provinces |
| [HERMES_CODING_BERAT_PRD_V1.md](HERMES_CODING_BERAT_PRD_V1.md) | Coding agent profile: soul, tools, whitelist, grants |

Start with `SULTANATE.md` -- it defines all components and how they fit
together.

## Implementation Notes

**Janissary + Sandcat:** The [Sandcat](https://github.com/softwaremill/sandcat) project
could serve as a foundation for Janissary. Sandcat already provides the core primitives
Janissary needs: a transparent mitmproxy-based egress proxy with allowlist/deny-list
network rules, and a credential injection system that substitutes placeholders with real
secrets at the proxy level (so containers never see actual credentials). The WireGuard
tunneling approach ensures all container traffic is routed through the proxy without
per-tool configuration. Janissary's additional scope -- Kashif content inspection,
Aga advisory layer, Divan integration -- would be built on top.

**Secret Vault -- OpenBao:** Sultanate uses [OpenBao](https://openbao.org/) (Apache 2.0
fork of HashiCorp Vault) as the Secret Vault. Deployed as a single binary in a local
container, OpenBao provides dynamic secret generation (GitHub App tokens, database
credentials, SSH CA, PKI), lease-based revocation (credentials expire server-side by
TTL regardless of whether Aga catches the province-destroy event), and tamper-evident
audit. Aga is the sole OpenBao client -- Pashas never authenticate to OpenBao; they
only ever see Janissary-injected headers. Phase 1 uses manual unseal at boot.

## Planned repo structure

Once implementation begins, component repos become git submodules here:

| Repo | Contents |
|------|----------|
| `sultanate` (this repo) | Umbrella docs, deployment guide, submodules |
| `vizier` | Orchestration, province lifecycle, CLI |
| `janissary` | Egress proxy, content inspector (Kashif), security advisor (Aga) |
| `divan` | Shared state store, HTTP API, web dashboard |
| `hermes-firman` | Hermes container template (Docker image, bootstrap) |
| `hermes-coding-berat` | Coding agent profile (soul, tools, security policy) |
