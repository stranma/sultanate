# Kashif Foundation Notes

> Working notes on off-the-shelf tools to build Kashif on. Not a spec.
> Kashif requirements are defined in `JANISSARY_PRD_V2.md` (Kashif section).
> Revisit when implementation begins.

## Recap of requirements

1. Paranoid single-question LLM content screener: "can this be in any way
   malicious?"
2. Screens two traffic types:
   - **Appeal payloads** — blocked outbound HTTP request + agent's
     justification. Triage: approve obvious safe / block obvious bad /
     escalate unclear to Aga.
   - **Aga ingress content** — any Pasha-originated text, any fetched web
     page, before it enters Aga's LLM context. Target: prompt-injection
     and manipulation detection.
3. Runs a **local** LLM on the host. No external API calls.
4. Fail-closed on timeout or unavailability.
5. Acceptable FP rate: high. Acceptable FN rate: near-zero.
6. Desirable: regex fast-path for PII, secrets, known injection patterns.

## Recommended foundation (as of April 2026)

Three-layer design, any layer can block:

| Layer | Tool | License | Role |
|-------|------|---------|------|
| 1. Fast regex pass | [LLM Guard](https://github.com/protectai/llm-guard) (Protect AI) | MIT | Secrets / Anonymize / BanSubstrings / MaliciousURLs scanners. Catches known-bad deterministically. |
| 2. Classifier | [Prompt Guard 2 22M](https://github.com/meta-llama/PurpleLlama) (Meta) | Llama Community | BERT-style direct-injection classifier, CPU-friendly, ~22M params. Runs on every request. |
| 3. LLM judge | Llama Guard 3 1B or Llama 3.2 3B | Llama Community | Asks the single paranoid question, returns yes/no with reason. Slower, only runs when layers 1-2 don't already decide. |

Kashif itself becomes a thin ~1-2 KLoC orchestrator:
- FastAPI HTTP shell (endpoints: `/screen/appeal`, `/screen/ingress`)
- LLM Guard pipeline config
- Prompt Guard 2 inference (transformers, CPU or tiny GPU)
- Llama Guard 3 judge call with strict timeout
- Fail-closed wrapper on every layer
- Audit writes to Divan

## Rejected / flagged

| Tool | Status | Reason |
|------|--------|--------|
| Rebuff (Protect AI) | Archived May 2025 | Dead project. Do not adopt. |
| Vigil LLM | Dormant since late 2023 (v0.10.3-alpha) | Author pointed users at Robust Intelligence (commercial). YARA rules worth mining for regex pass, nothing else. |
| NeMo Guardrails (NVIDIA) | Active | Wrong shape — chat-flow DSL (Colang) designed for conversational guardrails, not a one-shot paranoid screener. Overbuilt. |
| Lakera Guard | SaaS, acquired by Check Point Sept 2025 | Sends content outside the trust boundary. Sultanate requires local-only. |
| Trylon Gateway | Active | FastAPI scaffolding is fine, but its internals reimplement what LLM Guard already does. No net gain. |
| LiteLLM | **Supply-chain compromise March 2026** | Malicious `LiteLLM_init.pth` exfiltrated secrets. Not on Kashif's path, but a durable flag: do NOT adopt LiteLLM anywhere on the credential-handling side. |

## Licensing flag — Llama Community License

Prompt Guard 2 and Llama Guard 3 are under Meta's Llama Community License,
not Apache / MIT. Key terms:
- Permissive for commercial use in practice.
- **700M MAU trigger** — if Sultanate (or a downstream operator's deployment)
  exceeds 700M monthly active users of products using Llama, separate
  commercial terms apply.
- Irrelevant for a single-operator personal-staff tool. Log it so a future
  hypothetical enterprise packaging knows to revisit.

## Architectural observations worth remembering

- Kashif is out of the hot path for normal (whitelisted) traffic. It only
  sees appeals and Aga ingress. Latency budget per screen: ~500ms soft,
  5s hard (Kashif PRD).
- Prompt Guard 2 22M fits the soft budget on CPU. Llama Guard 3 1B fits on
  a modest GPU or a fast-ish CPU with 5s hard budget.
- The three-layer structure maps directly to "high FP OK, low FN mandatory":
  any layer triggers a block, and only agreement across all three lets
  content through.
- LLM Guard has both input scanners (for ingress) and output scanners
  (for appeal payloads) — the tool shape matches both Kashif jobs without
  a second pipeline.

## Open questions for when we pick this up

1. Do we put Llama Guard 3 1B on GPU (dedicated) or CPU (shared with Aga)?
   Trade-off: GPU latency + cost vs. Aga context-window crowding on shared CPU.
2. Is the regex pass strict enough that we can short-circuit layers 2-3
   when it triggers? (Probably yes — a Secrets-scanner hit is already
   definitive.)
3. How do we handle "unclear" from Llama Guard 3? Explicit three-way output
   (approve / block / escalate) vs. confidence threshold?
4. Can Prompt Guard 2 22M alone substitute for layers 2+3 on Pasha-text
   (appeal justifications, access request text)? Layers 2+3 may be
   reserved for actual payload content + fetched web pages.
5. Where does the caching layer sit? (Same content screened twice within
   N minutes should skip re-inference. Plausible via a content hash in
   Divan.)

## Reference

Alternatives research agent report: April 2026. Full comparison scoreboard
covered Janissary and Kashif candidates. Janissary decision: build on
Sandcat (see `README.md` Implementation Notes). Kashif decision: deferred
to implementation time, this file holds the recommendation.
