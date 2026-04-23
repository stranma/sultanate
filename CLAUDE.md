# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo status

**Design phase — PRDs only, no code, no build system.** Every file in the working tree is Markdown. There is nothing to build, lint, or test. Do not invent commands or scaffolding; if a task needs executable code, the answer is usually "that belongs in a component submodule that does not exist yet."

Planned submodule layout (not yet created) lives in `README.md` and `SULTANATE.md` — `vizier/`, `janissary/` (contains Kashif + Aga + Divan), `hermes-firman/`, `hermes-coding-berat/`.

## Document hierarchy

`SULTANATE.md` is the umbrella. Every other PRD declares itself subordinate with this line at the top:

> For shared glossary, deployment model, and component overview see [SULTANATE.md](SULTANATE.md).

**Always consult `SULTANATE.md` first** for terminology, trust model, network topology, and cross-component contracts. Component PRDs assume that shared context and do not restate it. When editing any component PRD, check `SULTANATE.md` for the authoritative definition before introducing or renaming a concept.

Reading order for a cold start:
1. `README.md` — index, current status, Sandcat/SentinelGate notes
2. `SULTANATE.md` — architecture, Ottoman glossary, trust model, failure modes
3. Component PRDs: `JANISSARY_PRD_V2.md`, `VIZIER_PRD_V3.md`, `HERMES_FIRMAN_PRD_V1.md`, `HERMES_CODING_BERAT_PRD_V1.md`
4. `SENTINELGATE_ANALYSIS.md` — integration analysis, not a spec; captures which SentinelGate capabilities Sultanate plans to adopt vs. reject

## Architecture invariants

These hold across every component PRD. Violating them in a doc edit is almost always wrong:

- **Ottoman naming is load-bearing, not decorative.** The metaphor encodes the trust hierarchy (Sultan → trusted core → untrusted Pashas) and the governance model. See the glossary table in `SULTANATE.md`. Use the Ottoman term (Pasha, Province, Firman, Berat, Divan, Realm, etc.) consistently — introducing alternate names ("agent", "container", "template") inside PRD prose blurs the model. First use per document may gloss the term, e.g. "Pasha (agent inside a province)".
- **One-way dependencies via Divan.** No component calls another directly. Vizier writes province state → Divan; Aga reads Divan and reacts; Janissary reads Divan rules. If a proposed flow has component A calling component B, route it through Divan instead.
- **Janissary is dumb and has no outbound of its own.** No LLM, no content evaluation, no internet access beyond forwarding. Deterministic rule application only. Intelligence lives in Kashif (paranoid LLM screener) and Aga (trusted advisor agent).
- **Fail-closed everywhere.** Divan unreachable → Janissary uses last-cached rules; no cache → block all. Kashif unresponsive → block. Never propose "fail open" or "pass-through on degraded service" behavior.
- **Aga is trusted but guarded.** All Aga inputs — appeal justifications, fetched web content, access request text — are pre-screened by Kashif for prompt injection before reaching Aga's context. Aga cannot expand its own whitelist; only Sultan can.
- **Network-level enforcement is unbypassable; application-level is not.** Janissary is the network route, not a tool agents call. The only application-level touchpoint is the Janissary security MCP (`appeal_request`, `request_access`).
- **Province state ≠ task state.** Provinces are long-lived; `creating / running / stopped / failed / destroying` is infrastructure lifecycle. Task state is a Phase 3 concern and does not live in Vizier or the province registry.
- **Phase 1 is Hermes-native and single-host.** Runtime-agnostic berats, OpenClaw support, and multi-machine are explicitly deferred. Do not add them to Phase 1 scope sections.
- **OpenBao is the Secret Vault; Aga is its sole client.** Pashas never authenticate to OpenBao — they only see Janissary-injected headers. Every credential is lease-bound so that missed revocations on province destroy are bounded by TTL, not by Aga's reliability. Manual unseal at boot in Phase 1. When editing PRDs, the product name is **OpenBao** and the role name is **Secret Vault** (parallel to Divan-the-role / SQLite-the-implementation).
- **MVP threat model: the only adversary in scope is a hostile Pasha.** Trusted-core components (Aga, Janissary, Kashif, Divan, OpenBao, host) are assumed uncompromised. Do not propose hardening against trusted-core compromise — signed audit chains, signed-manifest cross-checks between Janissary and Aga, Aga-AppRole rotation on restart, multi-operator Shamir, dual audit sinks with divergence monitoring, etc. — unless the user specifically asks.

## Editing conventions

- PRDs use ATX headings, 80-column-ish wrapping, and tables for structured facts (parameters, permissions, component responsibilities). Match the surrounding style when adding sections.
- When adding a concept used by multiple components, put the canonical definition in `SULTANATE.md` and reference it from the component PRD — do not duplicate definitions across files.
- Each component PRD has a `Phase 1 Scope` section with `In scope` / `Deferred` subsections. Keep them consistent with `SULTANATE.md`'s `Phase 1 Scope` — if they diverge, `SULTANATE.md` wins and the component PRD needs updating.
- Integration analyses (like `SENTINELGATE_ANALYSIS.md`) are exploratory, not normative. Conclusions drawn there only become binding once folded into `SULTANATE.md` or a component PRD.
