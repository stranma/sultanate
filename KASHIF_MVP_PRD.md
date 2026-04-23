# PRD: Kashif MVP -- Content Inspector for Sultanate

> For shared glossary and architecture see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For the proxy sibling see [JANISSARY_MVP_PRD.md](JANISSARY_MVP_PRD.md).
> For shared state contract see [DIVAN_API_SPEC.md](DIVAN_API_SPEC.md).
> Working notes that preceded this PRD: [KASHIF_NOTES.md](KASHIF_NOTES.md).

## What Kashif Is

A paranoid single-question content screener running three model layers on
CPU, all local. It answers exactly one question: *"Can this content be in
any way malicious?"* Any layer can block; content must pass all three to
be allowed. Accepts high false-positive rate; does not accept false
negatives.

Kashif ships with Janissary in the `janissary` repo but runs as a separate
container. Janissary is deterministic and dumb; Kashif is the LLM judge.

## What Kashif Screens

1. **Appeal payloads (`/screen/appeal`)** -- when Janissary blocks a
   write request and the agent appeals, the full request body plus the
   agent's justification text are sent to Kashif for triage. Returns
   `allow` / `block` / `escalate`.

2. **Aga ingress (`/screen/ingress`)** -- any Pasha-originated text
   headed for Aga's LLM context (access-request justification, freeform
   Telegram text from a Pasha, access request reasoning), plus any web
   content Aga fetches that is about to enter its context. Returns
   `allow` / `block` / `escalate`. Fetched web content is handled via
   the same endpoint with `source: "web"`.

Kashif is **not** on the hot path for normal browsing traffic. It only
processes appeals and trust-boundary ingress. Whitelisted traffic
bypasses Kashif entirely.

## Three-Layer Pipeline

Any layer can return a definitive verdict. All three must agree to
`allow`.

| Layer | Tool | License | Role | Typical latency (CPU) |
|-------|------|---------|------|-----------------------|
| 1. Regex fast-path | [LLM Guard](https://github.com/protectai/llm-guard) scanners | MIT | Secrets / Anonymize / BanSubstrings / MaliciousURLs. Deterministic, known-bad patterns. | ~10 ms |
| 2. Classifier | [Prompt Guard 2 22M](https://github.com/meta-llama/PurpleLlama) | Llama Community | BERT-style direct-injection classifier, ~22M params. | ~50-200 ms |
| 3. LLM judge | Llama Guard 3 1B (Q4 quantization) | Llama Community | Asks the single paranoid question. Returns yes/no with a short reason. | ~1-2 s |

Hard timeout: **5 s** total. If any layer is slow or the pipeline exceeds
budget, verdict is `escalate` with `reason: timeout`.

## Endpoints

```
POST /screen/appeal      (Janissary role key)
POST /screen/ingress     (Janissary, Aga role keys)
GET  /health             (no auth)
```

### `POST /screen/appeal`

Request (from Janissary):
```json
{
  "appeal_id": "appeal-m1n2o3",
  "url": "https://pastebin-clone.xyz/upload",
  "method": "POST",
  "payload": "<full request body, any length>",
  "justification": "Uploading test failures for debugging",
  "province_id": "prov-a1b2c3"
}
```

Response (success, within timeout):
```json
{
  "verdict": "allow",
  "reason": "regex clean; prompt-guard-2 score 0.02; llama-guard-3 benign",
  "screened_at": "2026-04-23T11:00:02Z",
  "latency_ms": 1420
}
```

`verdict` enum: `allow`, `block`, `escalate`.

Kashif also writes the verdict directly to Divan via
`PATCH /appeals/{appeal_id}/kashif_verdict` (Kashif has the `kashif`
role key). Divan auto-transitions the appeal based on the verdict:

- `allow` -> `status: approved, decision: one-time` + audit `severity=info`
- `block` -> `status: denied` + audit `severity=alert`
- `escalate` -> stays `pending` + audit `severity=alert`

Kashif's HTTP response to Janissary is informational only; the
authoritative state lives in Divan.

### `POST /screen/ingress`

Request (from Janissary or Aga):
```json
{
  "content": "<free-form text or fetched HTML>",
  "source": "pasha",
  "province_id": "prov-a1b2c3",
  "context": {
    "purpose": "access_request_justification"
  }
}
```

`source` enum: `pasha` (originated by a Pasha, e.g., appeal text or
access-request justification), `web` (content Aga fetched and is about
to ingest).

Response:
```json
{
  "verdict": "allow",
  "reason": "regex clean; prompt-guard-2 score 0.01; llama-guard-3 benign",
  "screened_at": "2026-04-23T11:05:00Z",
  "latency_ms": 980
}
```

Caller (Janissary or Aga) is expected to honour the verdict: on
`block`, do not forward the content to Aga; on `escalate`, forward with
a flag indicating Aga should surface the content to Sultan before
acting on it.

Kashif writes an audit entry to Divan for every ingress screen
(severity=info on allow, alert on block/escalate).

### `GET /health`

No authentication. Returns `200` when all three model layers are
resident and responsive.

```json
{
  "status": "ok",
  "models": {
    "llm_guard": "ready",
    "prompt_guard_2": "ready",
    "llama_guard_3": "ready"
  },
  "uptime_seconds": 1823
}
```

Returns `503` during cold start while models are loading, or if any
model is unresponsive.

## Fail-Closed Semantics

| Condition | Verdict | Reason |
|-----------|---------|--------|
| Any layer crashes mid-request | `escalate` | `reason: layer_error` |
| Total pipeline >5 s | `escalate` | `reason: timeout` |
| Kashif container unreachable (from Janissary's perspective) | Janissary writes `kashif_verdict=escalate` itself with `kashif_timeout=true` |
| Kashif HTTP 5xx response | `escalate` | caller treats as unavailability |

Never `allow` on failure. Never `block` on failure (a failure is not
evidence of malice). Escalate so Sultan sees the appeal/ingress with
the failure context and decides.

## Resource Budget

Target: Hetzner AX41-NVMe (Ryzen 5 3600, 64 GB RAM, no GPU).

| Component | RAM resident | Notes |
|-----------|--------------|-------|
| LLM Guard scanners (fast-path) | ~500 MB | Mostly DeBERTa-small for the prompt-injection scanner; the rest are regex and small classifiers. |
| Prompt Guard 2 22M | ~100 MB | BERT-style, CPU-friendly. |
| Llama Guard 3 1B (Q4) | ~1.5 GB | GGUF Q4 via llama-cpp-python. |
| FastAPI + runtime | ~300 MB | Python 3.12, httpx, uvicorn. |
| **Total** | **~2.5-4 GB** | Fits alongside Janissary, Aga, Vizier, OpenBao, Divan. |

CPU: dedicate 2 threads to Kashif under load (Llama Guard 3 1B Q4 on
CPU is ~1-2 s per judge call). Since Kashif is off the hot path
(appeals + ingress only, not every request), sustained throughput is
not a concern in MVP.

## Models Baked Into Docker Image

Model weights are downloaded at image build time, not at container
start. First boot does not need external network access for model
pulls. Trade-off: image size grows by ~2.5 GB.

```dockerfile
# kashif/Dockerfile (excerpt)
RUN pip install --no-cache-dir \
    fastapi==0.115.* \
    uvicorn[standard]==0.34.* \
    httpx==0.28.* \
    llm-guard==0.3.* \
    transformers==4.45.* \
    llama-cpp-python==0.3.*

# Download weights during build
RUN python -c "from llm_guard.input_scanners import ...; \
    <pre-download LLM Guard scanner models>"

RUN python -c "from transformers import AutoTokenizer, AutoModel; \
    AutoTokenizer.from_pretrained('meta-llama/Prompt-Guard-2-22M'); \
    AutoModel.from_pretrained('meta-llama/Prompt-Guard-2-22M')"

# Llama Guard 3 1B Q4 GGUF
RUN curl -L -o /opt/kashif/models/llama-guard-3-1b-q4.gguf \
    https://huggingface.co/meta-llama/Llama-Guard-3-1B/resolve/main/llama-guard-3-1b-Q4_K_M.gguf

ENV KASHIF_MODELS_DIR=/opt/kashif/models
EXPOSE 8082
ENTRYPOINT ["python", "/opt/kashif/main.py"]
```

The Llama Community License requires acceptance before downloading Meta
models. The deploy script prompts the operator to accept the license
once; the acceptance flag is persisted in the image-build pipeline
config.

## Configuration

`/opt/sultanate/kashif/config.yaml`:

```yaml
server:
  host: "127.0.0.1"
  port: 8082

divan:
  url: "http://127.0.0.1:8600"
  api_key_env: "DIVAN_KEY_KASHIF"

pipeline:
  timeout_seconds: 5                 # hard total budget

  llm_guard:
    enabled: true
    scanners:
      - "Secrets"
      - "Anonymize"
      - "BanSubstrings"
      - "MaliciousURLs"
    fail_fast: true                  # first hit returns block

  prompt_guard_2:
    enabled: true
    model_path: "/opt/kashif/models/prompt-guard-2-22m"
    threshold: 0.5                   # score above = block

  llama_guard_3:
    enabled: true
    model_path: "/opt/kashif/models/llama-guard-3-1b-q4.gguf"
    ctx_size: 4096
    n_threads: 2                     # CPU threads for llama.cpp

pre_warm: true                       # load all models at startup
                                     # (readiness delayed by ~10-30 s
                                     #  but every request is fast)
```

## Divan Integration

Kashif writes to Divan:

- `PATCH /appeals/{appeal_id}/kashif_verdict` -- Kashif role only.
  Writes the verdict after appeal screening. Divan auto-transitions
  appeal status based on verdict.
- `POST /audit` -- Kashif role. Every screen call appends an audit
  entry (severity info/alert). Content of the screened payload is
  NOT stored in audit; only the verdict, reason, and metadata.

Kashif reads no state from Divan. It is stateless between requests
(no history, no province-specific configuration). The three models are
the entire decision surface.

## Deployment

### docker-compose.yml entry

```yaml
kashif:
  build:
    context: ./kashif
  depends_on:
    divan:
      condition: service_healthy
  network_mode: host                 # 127.0.0.1:8082 only
  volumes:
    - /opt/sultanate/kashif/config.yaml:/opt/kashif/config.yaml:ro
    - /opt/sultanate/divan.env:/opt/sultanate/divan.env:ro
  env_file:
    - /opt/sultanate/divan.env       # provides DIVAN_KEY_KASHIF
  restart: unless-stopped
  healthcheck:
    test: ["CMD", "curl", "-sf", "http://127.0.0.1:8082/health"]
    interval: 5s
    timeout: 3s
    retries: 30                      # slow cold start (model load)
```

### Startup

1. Load config from `/opt/kashif/config.yaml`.
2. Pre-warm: import each model class, run one dummy inference to trigger
   lazy initialization (LLM Guard scanners, Prompt Guard 2 via
   transformers, Llama Guard 3 via llama-cpp-python). Total ~10-30 s on
   Ryzen 5 3600.
3. Bind FastAPI to `127.0.0.1:8082`.
4. `/health` returns `200` when all three models report ready.

### Graceful shutdown

On `SIGTERM`:
1. FastAPI stops accepting new connections
2. In-flight screens complete (5 s max)
3. Models unload (process exit releases RAM)

No persistent state. On restart, cold boot repeats the model pre-warm.

## What Kashif Does NOT Do

- **No memory / no history.** Every screen call is independent. Pattern
  detection across calls is Aga's job, not Kashif's.
- **No policy authorship.** Kashif answers "malicious?" with yes/no;
  it does not write rules, manage blacklists, or modify whitelists.
  Aga handles all of that.
- **No Sultan interaction.** Kashif has no Telegram channel, no direct
  operator surface. Verdicts reach Sultan only indirectly via Divan
  audit + Vizier/Aga polling.
- **No browsing of web content on its own.** Only screens content
  posted to it by Janissary or Aga.
- **No hot-path involvement.** Whitelisted traffic never touches Kashif.

## Phase 1 Scope

**In scope:**
- Three-layer pipeline (LLM Guard regex, Prompt Guard 2 22M, Llama
  Guard 3 1B Q4), all CPU
- `/screen/appeal`, `/screen/ingress`, `/health` endpoints
- Writes kashif_verdict to Divan
- Writes audit entries to Divan with severity
- Fail-closed on timeout or layer error
- Pre-warm at startup (models resident before /health goes 200)
- Weights baked into Docker image (no first-run download)

**Deferred:**
- Per-province screening tuning (all provinces share the same
  thresholds)
- Llama Guard 3 8B (requires GPU)
- Local Ollama integration (current design uses transformers +
  llama-cpp-python directly; Ollama is a deployment detail)
- Rate limiting / quota per Pasha (Aga's job via audit pattern
  detection)
- Adversarial-robustness testing harness for the classifier models
- Phase 2 option: signed verdicts (ECDSA per-entry) for tamper-
  evident Kashif audit
- Caching identical content hashes to skip re-inference
- Evaluation harness + red-team corpus for FP/FN measurement (a
  real gap -- the "low FN mandatory" requirement is untestable
  without one; Phase 2)
