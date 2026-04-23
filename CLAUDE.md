# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo status

**Design phase — PRDs and technical SPECs only, no code, no build system.** Every file in the working tree is Markdown. There is nothing to build, lint, or test. Do not invent commands or scaffolding; if a task needs executable code, the answer is usually "that belongs in a component submodule that does not exist yet."

Planned submodule layout (not yet created) lives in `README.md` and `SULTANATE_MVP.md` — `vizier/`, `janissary/` (contains Kashif, Aga, and Divan), `openclaw-firman/`, `openclaw-coding-berat/`.

## Document hierarchy

`SULTANATE_MVP.md` is the umbrella for the active MVP. Every component PRD declares itself subordinate with this line at the top:

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).

**Always consult `SULTANATE_MVP.md` first** for terminology, trust model, network topology, and cross-component contracts. Component PRDs assume that shared context and do not restate it. When editing any component PRD, check `SULTANATE_MVP.md` for the authoritative definition before introducing or renaming a concept.

Reading order for a cold start:

1. `README.md` — doc index grouped by layer (cross-cutting, security, orchestration, runtime, exploratory)
2. `MOTIVATION.md` — why the system exists (problem statement, operator goals)
3. `ARCHITECTURE.md` — system + flow diagrams, 10 user stories with test assertions
4. `SULTANATE_MVP.md` — MVP scope, trust model, credential model, startup order, hardware target
5. Cross-component contracts:
   - `DIVAN_API_SPEC.md` — shared-state HTTP API every component depends on
6. Security perimeter (one repo: `janissary`):
   - `JANISSARY_MVP_PRD.md` + `JANISSARY_SPEC.md`
   - `KASHIF_MVP_PRD.md`
   - `AGA_MVP_PRD.md` + `AGA_SPEC.md`
   - `DIVAN_MVP_PRD.md`
7. Orchestration:
   - `VIZIER_MVP_PRD.md` + `VIZIER_SPEC.md`
8. Province runtime (OpenClaw, Phase 1):
   - `OPENCLAW_FIRMAN_MVP_PRD.md` + `OPENCLAW_FIRMAN_SPEC.md`
   - `OPENCLAW_CODING_BERAT_MVP_PRD.md` + `OPENCLAW_CODING_BERAT_SPEC.md`
9. `SENTINELGATE_ANALYSIS.md` — integration analysis, not a spec; captures which SentinelGate capabilities Sultanate plans to adopt for Phase 2 (session tracking, tool-level RBAC, ECDSA-signed audit)

The `origin/archive-hermes-infisical` branch preserves the pre-MVP Hermes + Infisical + Sentinel baseline — useful as historical reference but not the authoritative source.

## Architecture invariants

These hold across every component PRD. Violating them in a doc edit is almost always wrong:

- **Ottoman naming is load-bearing, not decorative.** The metaphor encodes the trust hierarchy (Sultan → trusted core → untrusted Pashas) and the governance model. See the glossary section in `SULTANATE_MVP.md`. Use the Ottoman term (Pasha, Province, Firman, Berat, Divan, Realm, Aga) consistently — introducing alternate names ("agent", "container", "template") inside PRD prose blurs the model. First use per document may gloss the term, e.g. "Pasha (agent inside a province)".
- **One-way dependencies via Divan.** No component calls another directly. Vizier writes province state → Divan; Aga reads Divan and reacts; Janissary reads Divan rules; Kashif writes appeal verdicts to Divan. If a proposed flow has component A calling component B, route it through Divan instead.
- **Janissary is dumb and has no outbound of its own.** No LLM, no content evaluation, no internet access beyond forwarding. Deterministic rule application only. Intelligence lives in Kashif (paranoid LLM screener) and Aga (trusted advisor agent).
- **Fail-closed everywhere.** Divan unreachable → Janissary uses last-cached rules; no cache → block all. Kashif unresponsive or timed out → treat as `escalate` (never auto-approve). OpenBao sealed → Aga cannot mint new tokens; existing valid leases keep working; expired leases fail closed on Janissary injection. Never propose "fail open" or "pass-through on degraded service" behavior.
- **Aga is trusted but guarded.** All Aga ingress — appeal justifications, fetched web content, access request text, any Pasha-originated text — is pre-screened by Kashif for prompt injection before reaching Aga's LLM context. Aga cannot expand its own whitelist; only Sultan can.
- **Network-level enforcement is unbypassable; application-level is not.** Janissary is the network route, not a tool agents call. The only application-level touchpoint is the Janissary security MCP (`appeal_request`, `request_access`).
- **Province state ≠ task state.** Provinces are long-lived; `creating / running / stopped / failed / destroying` is infrastructure lifecycle. Task state is a Phase 3 concern and does not live in Vizier or the province registry.
- **Phase 1 runtime is OpenClaw.** The only firman is `openclaw-firman`; the only berat is `openclaw-coding-berat`. Additional runtimes (OpenHands, CrewAI, custom) are Phase 2 firmans, not a Phase 1 concern.
- **OpenBao is the Secret Vault; Aga is its sole client.** Pashas never authenticate to OpenBao — they only see Janissary-injected headers. In Phase 1, the primary credential path is **GitHub App dynamic minting**: Sultan sets up the GitHub App once, Aga mints per-province installation tokens with 1-hour TTL and auto-renews every ~15 min while the province is running. KV fallback exists for services without a dynamic mint path. When editing PRDs, the product name is **OpenBao** and the role name is **Secret Vault**.
- **MVP threat model: the only adversary in scope is a hostile Pasha.** Trusted-core components (Aga, Janissary, Kashif, Divan, OpenBao, host) are assumed uncompromised. Do not propose hardening against trusted-core compromise — signed audit chains, signed-manifest cross-checks between Janissary and Aga, Aga-AppRole rotation on restart, multi-operator Shamir, dual audit sinks with divergence monitoring, etc. — unless the user specifically asks.
- **Target host: Hetzner AX41-NVMe (Ryzen 5 3600, 64 GB RAM, no GPU).** Kashif runs CPU-only (LLM Guard regex + Prompt Guard 2 22M + Llama Guard 3 1B Q4). Aga and Pashas call Claude Sonnet / OpenRouter remotely via Janissary. GPU upgrade is Phase 2.

## Kashif verdict → Divan auto-transition

Kashif writes `kashif_verdict` to Divan on each appeal screening:

- **allow** → Divan auto-transitions appeal to `approved` (one-time). Audit severity `info`. Sultan not notified. Silent path.
- **block** → Divan auto-transitions to `denied`. Audit severity `alert`. **Both Sultan and Aga are notified** (informational) via their own polling — so they can spot Pasha drift (e.g., repeated Kashif blocks mean "consider destroying the province").
- **escalate** (or Kashif timeout / unavailable) → appeal stays `pending`. Audit severity `alert`. Vizier relays actionable Telegram to Sultan; Aga adds behavioural context.

Only the Kashif=allow branch is silent. Both block and escalate reach the operator. Do not regress this in PRD edits.

## Editing conventions

- PRDs use ATX headings, 80-column-ish wrapping, and tables for structured facts (parameters, permissions, component responsibilities). Match the surrounding style when adding sections.
- When adding a concept used by multiple components, put the canonical definition in `SULTANATE_MVP.md` and reference it from the component PRD — do not duplicate definitions across files.
- Each component PRD has a `Phase 1 Scope` section with `In scope` / `Deferred` subsections. Keep them consistent with `SULTANATE_MVP.md`'s `What's Deferred` list — if they diverge, `SULTANATE_MVP.md` wins and the component PRD needs updating.
- SPEC docs (`JANISSARY_SPEC.md`, `AGA_SPEC.md`, `VIZIER_SPEC.md`, `OPENCLAW_*_SPEC.md`, `DIVAN_API_SPEC.md`) contain implementation detail (schemas, code shapes, Docker commands). Keep them concrete; link back to the matching MVP PRD for scope.
- Integration analyses (like `SENTINELGATE_ANALYSIS.md`) are exploratory, not normative. Conclusions drawn there only become binding once folded into `SULTANATE_MVP.md` or a component PRD.
- File naming: MVP scope docs end in `_MVP_PRD.md`; technical specs end in `_SPEC.md`. Exceptions: `SULTANATE_MVP.md` (umbrella, no PRD suffix), `MOTIVATION.md`, `ARCHITECTURE.md`, `README.md`, `CLAUDE.md`, `SENTINELGATE_ANALYSIS.md` (analysis).
