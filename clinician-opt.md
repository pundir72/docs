# Clinician Agent: issues, cleanup and AI cost optimization

Reviewed: 9 September 2026. Source review plus official Google pricing/lifecycle documentation. No production traffic, billing export, benchmark or live API test was available. Findings below distinguish visible code defects from risks needing reproduction. Application code has not been changed.

**Recommended order:** protect report correctness → fix model availability and concurrency → stop unnecessary AI work → simplify code → measure and tune infrastructure. Making an incorrect save faster is not a useful optimization.

## 1. Fix first: correctness and reliability

| Priority | Issue and evidence | Impact | How to fix / verify |
|---|---|---|---|
| P0 | Normal `_process_form_mode` sends the whole transcript to plan processing and leaves other segments empty. [Agent](Clinician-agent/src/llm/functionalities.py) | Objective/subjective/RPE voice instructions can fail to update their intended section. | Introduce explicit form/section routing and typed operations for supported sections. Test each form, negations, left/right, units and edits; reject unsupported operations explicitly. |
| P0 | Manual and auto-save have different transformations; `PAYLOAD_BUILDERS` explicitly handles only assessment/firstAssessment. [API](clinician-agent-FE-v2/src/utils/api.ts), [manual save](clinician-agent-FE-v2/src/hooks/use-form-management.tsx) | Same visible form can produce different saved data; SNC/physio fall back to an assessment builder on the renderer path. | One canonical form model and one serializer per form type; one shared save service. Add contract fixtures for all four forms and verify external GraphQL input/update semantics. |
| P0 | Auto-save filters out `0`, `false`, empty strings and empty collections. [Submission](clinician-agent-FE-v2/src/utils/form-submission.ts) | Valid zero values or deliberate clearing may be lost. Actual persisted behavior depends on external merge rules. | Distinguish unchanged, set and clear operations. Preserve valid zero/false. Define explicit null/delete semantics with GraphQL owner; test clearing an existing value. |
| P0 | Authentication payload is acknowledged without key validation; browser includes static key values. [Socket endpoint](Clinician-agent/main.py), [socket client](clinician-agent-FE-v2/src/hooks/use-web-socket.ts) | No application-level proof of clinician access to the supplied patient/appointment; potentially uncontrolled paid calls. Infrastructure enforcement is unknown. | Verify existing gateway controls. Validate short-lived identity server-side and authorize appointment access. Treat browser keys as public; rotate if they were intended as secrets. Never trust supplied patient ID as authentication. |
| P1 | Auto-save timer is not assigned to its cancellation ref; 3000 ms prop is ignored in favor of 500 ms plus a 100 ms wait. [Auto-submit](clinician-agent-FE-v2/src/utils/auto-submit-manager.ts) | Cancellation can fail; delayed closures may submit stale data or duplicate saves. | Store/clear the timeout, use latest state, serialize saves and use a real debounce. Test rapid typing, cancellation, navigation and two consecutive AI updates. |
| P1 | Socket has no end-to-end request ID/form revision; transcription clears processing before form result. [Socket hook](clinician-agent-FE-v2/src/hooks/use-web-socket.ts) | New edits can occur while old work is running; stale followups/results may affect newer state. | Add `requestId`, `formRevision` and explicit stages (`transcribing`, `filling`, `ready`, `saving`). Discard or reconcile stale results; process each response once. |
| P1 | Cache key uses text/form hash but not identity, history, model or prompt version. [Redis services](Clinician-agent/src/redis_services.py) | A history-dependent result can be reused in a different context; prompt/model changes may return stale outputs. | Namespace by authenticated tenant/session, include relevant context/revision and prompt/schema/model versions. Cache only validated successes; disable current response reuse until correctness is established. |
| P1 | Validation wrappers catch errors and return input; parse failures can look like a successful unchanged form. [Validation](Clinician-agent/src/schemas/validation.py), [main](Clinician-agent/main.py) | Malformed output may reach auto-save, or failure can be presented as success. | Return typed success/no-change/validation-error states. Validate allowed paths and types before applying; suppress auto-save on errors; bounded repair only when useful. |
| P1 | Several modules log entire audio payloads, forms, transcripts and model prompts. [Socket hook](clinician-agent-FE-v2/src/hooks/use-web-socket.ts), [LLM manager](Clinician-agent/src/llm/llm_manager.py) | Sensitive data exposure plus browser, logging and trace-storage overhead. | Log request IDs, sizes, stages and timings by default; redact content; define retention and access controls. Sample sanitized diagnostic traces. |

P0 means address before expanding usage. P1 means prioritize during the next reliability iteration. This is a source-based prioritization, not a claim that every failure was reproduced.

## 2. Model availability: an immediate maintenance issue

The code sets `FAST_MODEL = gemini-2.0-flash-lite`. Google lists its retirement as **1 June 2026**. The configured `gemini-2.5-flash` and `gemini-2.5-pro` have listed retirement dates of **20 October 2026**. Google lists `gemini-3.1-flash-lite` as an upgrade for 2.0 Flash-Lite. Verify project/region availability and evaluate replacements now; do not add another soon-retiring model as a long-term fix. [Google model lifecycle](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-versions)

The existing fast route can attempt an unavailable model and then retry the primary. That can add latency even if failed requests are not billable. Actual request outcomes must be checked in logs; configuration alone does not show what production currently runs.

**Implementation:** put model IDs in validated configuration, create a separate immutable client for each route, maintain an availability check and migration date, and evaluate the replacement on the project's own examples before switching traffic.

## 3. Performance bottlenecks and fixes

| Bottleneck | Evidence | Optimization |
|---|---|---|
| Blocking speech processing | `process_audio_data` calls synchronous Pydub/Google STT inside an async handler. [main](Clinician-agent/main.py) | Run conversion/STT in a bounded executor or use an async speech client. Apply concurrency limits, request deadlines and audio size/duration limits. Moving to a thread alone is not backpressure. |
| Shared mutable model client | `complete_with_model` temporarily changes `self.model`; background tasks call the same singleton. [LLM manager](Clinician-agent/src/llm/llm_manager.py) | Separate clients per model, no shared mutation. A single Uvicorn worker still has concurrent async tasks/threads. Test simultaneous simple/complex requests. |
| Inconsistent pipeline branches | Text applies rate/cache/snapshot; audio+form bypasses them and duplicates followup logic. [main](Clinician-agent/main.py) | Normalize both inputs into one `process_instruction` service. Put admission control before STT/model costs and shared validation/telemetry around every route. |
| Unbounded followup work | `asyncio.create_task` results are not managed as a task group; old recommendations can outlive connection/revision. [main](Clinician-agent/main.py) | Track tasks by connection/revision, cancel obsolete work, cap concurrent calls and skip no-op/failed fills. Cancelling a thread wrapper may not cancel a provider request already in flight. |
| Inefficient history read | Redis does `LRANGE 0 -1`, then Python retains only 20 entries and plan uses five. [History](Clinician-agent/src/redis_handler.py) | Fetch the needed tail directly; trim lists and set intentional retention. Do not add LLM summarization unless it costs less than the repeated context it replaces. |
| Weak rate limiter | GET/check/INCR sequence is non-atomic; fail-open; keyed by client-supplied composite ID. [Services](Clinician-agent/src/redis_services.py) | Atomic Redis script/transaction with expiry, trusted tenant/user identity and per-tenant spend/concurrency limits; explicit degraded-mode policy. |
| Connection lifecycle inconsistencies | Client monitors every 300 seconds with an 80-second stale threshold; server stores ping_timeout but does not enforce it in its ping loop. [Client](clinician-agent-FE-v2/src/hooks/use-web-socket.ts), [server](Clinician-agent/main.py) | One heartbeat protocol, tracked reconnect timers, deliberate-close flag, cleanup on unmount and real pong deadlines. No reconnect after intentional close. |
| Broad React state updates | Large renderer state, deep clones, whole-form JSON logs and many callbacks. [Renderer](clinician-agent-FE-v2/src/components/forms/form-renderer.tsx) | Remove payload logs first, profile React, then memoize fields/stabilize callbacks and narrow state subscriptions. Virtualize only if measured long forms justify it. |
| Weak readiness/metrics | Health mostly checks object presence; route accounting is incomplete. [main](Clinician-agent/main.py) | Separate liveness from readiness, ping Redis with timeout, report queue depth/error rates. Avoid paid LLM calls on every health probe. |

Do not start by adding more workers or a larger VM. Fix shared-client races, blocking work and admission control first; otherwise replicas multiply the same costs and faults.

## 4. Where AI cost is currently wasted

### A. Followup calls on every form update

Normal successful audio+form work can perform:

```text
1 speech recognition operation (multiple chunks for long audio)
1 form-fill model call
1 recommendations model call
1 suggestions model call
```

The active page does not expose the recommendations value from the voice hook, despite paying for its generation. These extra model calls are launched even without evidence that the user needs them.

**Change:** do one fill call for a normal edit. Make recommendations explicit/on-demand if the product wants them. Generate suggestions at a meaningful pause or on request, only when form revision changed. Show deterministic missing-field messages without an LLM. Removing two calls reduces model-call count from three to one for affected interactions; it does **not** prove a 67% reduction in the total bill.

### B. Oversized static prompt

Measured directly from the active prompt strings:

| Prompt | Characters before form/history |
|---|---:|
| Full plan prompt | 28,326 |
| Fast plan prompt | 1,561 |
| Recommendation prompt | 1,861 |
| Suggestion prompt | 1,283 |

The full prompt is roughly 18 times the fast prompt's character count. Actual token counts need provider usage/tokenization; characters are not billed tokens. The plan path includes the full prompt directly, so the advertised context-manager budget does not cap this request.

**Change:** retain concise rules and a few representative examples, move deterministic field coercion to Python, send only relevant section data and necessary history. Version prompts and compare accuracy before removing examples that handle negation, units, repeated sets or edits.

### C. Output limits and structured responses

`MAX_OUTPUT_TOKENS` is 8192. A separate fast-manager factory defines 1024, but the active agent uses the shared default manager. A high maximum does not mean every request is billed for that maximum; actual generated output matters.

**Change:** configure an appropriate per-task cap, initially evaluated around 512–1024 tokens for small patches with explicit truncation detection. Use structured JSON response schemas to reduce prose/parsing repairs, then validate operations locally. Structured format does not establish clinical correctness. [Google structured output](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/capabilities/control-generated-output)

### D. Route based on measured quality, not only word count

Current routing uses text heuristics and switches a shared client. Use a supported economical model for evaluated extraction tasks; escalate once for explicit validation failure or known difficult categories. Do not escalate because the model claims low confidence alone. Do not silently move all failures to an expensive Pro model.

Direct UI actions such as clicking Remove or changing repetitions should remain ordinary code operations. Ambiguous clinical speech should not be handled by simplistic regex just to avoid a model call.

### E. Prevent repeat work before caching

Add an idempotency key for each recording/instruction and a request/revision-aware response protocol. Debounce duplicate processing and skip unchanged forms. Do not confuse a duplicate frontend callback with a proven second LLM request: measure network requests and provider calls separately.

Safe response caching requires all context that affects the output. For repeated static instructions, evaluate provider context caching after shortening prompts. Cache storage/minimum-size economics matter; small, low-reuse prompts may be cheaper uncached. [Google context caching](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/context-cache/context-cache-overview)

### F. Speech minutes can exceed the model bill

The code uses a V1-style Speech client. Trim leading/trailing silence without clipping words, reject empty audio and cap long recordings. Do not retranscribe audio when the user only edits the text. Measure the existing recorder's silence behavior before adding another detector.

Evaluate V2 as a separate migration, comparing terminology accuracy, accents, latency and API differences. Batch speech or batch model processing belongs to offline work, not the interactive response path. Do not enable data-logging discounts as an automatic cost optimization for patient audio.

## 5. Pricing and an illustrative budget

USD list prices checked on 9 September 2026; exclude infrastructure, taxes and negotiated discounts. Text input/output per million tokens: Gemini 2.5 Flash **$0.30/$2.50**, Pro **$1.25/$10** for short contexts; 3.1 Flash-Lite global **$0.25/$1.50**. Price alone is not a migration recommendation. [Google model pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing)

Speech V1 standard without data logging lists **$0.024/minute** after 60 free monthly minutes; V2 standard starts at **$0.016/minute**. Actual SKU, rounding, channels and account usage affect billing. [Google speech pricing](https://cloud.google.com/speech-to-text/pricing)

```text
LLM cost = sum across actual model calls:
  input_tokens / 1,000,000 × model_input_rate
  + billable_output_tokens / 1,000,000 × model_output_rate

Total = LLM + speech + Redis/compute + logs/traces + other cloud charges
```

**Illustration, not your measured bill:** 10,000 one-minute recordings/month; V1 without logging, 60-minute allowance available. Model work totals 10,000 input + 1,500 output tokens per interaction before, versus 1,500 + 300 afterward, all at Flash rates and no caching/retries. Output includes any billable reasoning.

| Monthly cost | Before | After |
|---|---:|---:|
| Model calls | $67.50 | $12.00 |
| Speech | $238.56 | $238.56 |
| Combined | $306.06 | $250.56 |

That is about **82% less model cost but only 18% less combined cost**. These calculations use the linked rate cards; token-volume reductions are assumptions. Measure audio minutes as well as model usage before promising savings.

## 6. Legacy code and what to do with it

| Candidate | Evidence/status | Cleanup approach |
|---|---|---|
| Root [functionalities.py](Clinician-agent/functionalities.py), [old_functionalities.py](Clinician-agent/src/llm/old_functionalities.py), [old_2_functionalities.py](Clinician-agent/src/llm/old_2_functionalities.py) | Not the active main import. | Preserve history in Git; verify test/script consumers, then remove duplicates from the working tree. |
| [old_prompts.py](Clinician-agent/src/old_prompts.py), [old_2_prompts.py](Clinician-agent/src/old_2_prompts.py) | Active agent imports prompts.py. | Remove after reference check; retain prompt versions in Git/evaluation artifacts. |
| Commented blocks in active agent | Active file has 3,907 lines, of which 2,348 are comment lines. | Remove historical code comments; keep useful explanations. Split agent into focused form, followup and validation services. Comments do not incur AI tokens unless included in a prompt. |
| [Alternative Vertex wrapper](Clinician-agent/src/llm/phoenix/vertex.py) | Active model integration is llm_manager.py. | Standardize on one provider adapter and telemetry layer; remove unused wrapper after dependency check. |
| [assesment-page.tsx](clinician-agent-FE-v2/src/pages/assesment-page.tsx), alternative assessment/physio renderers | Not the active App route/renderer. Some remain barrel exports. | Verify imports/exports, migrate any unique behavior, then remove unused alternatives. |
| `api.ts` and `graphql-client.ts` | Both have active callers and differing headers/serializers. | Consolidate deliberately into one transport plus query/mutation modules. Neither whole file is safe to delete immediately. |
| Multiple suggestion components and `index.ts` barrels | Similar names/case; some are active. | Keep one supported component, update consumers and remove unused exports. Barrel presence is not proof of screen usage. |
| RAG/vector constants, stubs and optional helpers | Active retrieval returns no chunks. | Remove unused APIs/config only after checking consumers; do not introduce a vector DB as an optimization of a feature not running. |
| Main Dockerfile's PyTorch/embedding prefetch | Leftover build work relative to current active pipeline. | Verify dependency use, use a minimal reproducible image and measure build/image size. Retain FFmpeg required for recording formats. |
| Jest config, Vitest config, npm/Bun locks, proxy alternatives | Competing tooling paths; npm scripts choose Vitest/Vite. | Agree on one package manager/test runner; remove unused config after CI/deployment checks. Audit direct/transitive dependencies before uninstalling. |
| Test logs/results, old schemas, audio management docs | Historical/reference artifacts; audio saving is commented out. | Move sanitized fixtures to tests, move bulky outputs out of source control and update docs. Keep fixtures used by tests. |

Removing legacy files mainly lowers maintenance/build burden. It does not automatically reduce model cost or the browser bundle: unused modules may already be excluded from production output.

## 7. Target design after cleanup

```text
Frontend
  one canonical form state + per-form schemas
  one GraphQL transport/save service
  one socket protocol with request ID, form revision and processing stage

Backend
  authenticate + authorize + validate + admit request
    → optional bounded STT
    → common instruction service for audio/text
    → deduplicate / context-correct cache
    → immutable model client + short structured prompt
    → validate allowed operations + apply patch
    → return versioned result
  optional followups only when requested

Browser save
  one serializer per form → ordered/version-aware GraphQL mutation
```

Keep the external GraphQL persistence boundary unless there is a separate product reason to change ownership. Any optimistic-concurrency/idempotency feature at that database boundary requires coordination with its API implementation.

## 8. Implementation sequence and acceptance checks

| Order | Work | Evidence required before moving on |
|---|---|---|
| 1 | Instrument request IDs, actual routed model/token usage, audio duration, stage latency, retries and saves. | Per-interaction traces reconcile approximately with provider billing; no raw patient payloads needed. |
| 2 | Fix model lifecycle, client mutation, save serializer parity, zero/clear semantics and timer/state races; verify authorization. | Contract tests for four forms; rapid-edit/out-of-order/cancel tests; simultaneous model-route test. |
| 3 | Unify audio/text admission and processing; move blocking STT; bound followups and provider calls. | Concurrent sessions do not starve heartbeats; timeouts/cancellation recover cleanly; no duplicate save after retry. |
| 4 | Remove automatic unused recommendations; gate suggestions; shorten prompts; tune output cap/model routing. | Same representative evaluation set meets agreed quality threshold; lower measured cost per accepted update. |
| 5 | Remove verified-unused files/dependencies/build steps; improve frontend rendering only where profiled. | Build/test/deployment entry points still work; bundle/image/latency measurements support the changes. |

Create an evaluation set covering all form types, multiple sets, units, left/right, negation, corrections, removal, zero/clear, noisy speech, invalid JSON and empty input. Track exact field correctness and clinician corrections, not just valid JSON.

Measure p50/p95 transcription time, fill time, time to confirmed save, successful saves, incorrect fields, stale updates, model calls/input/output per accepted update, retry rate, audio billed minutes and safe-cache hit rate. The active routed call uses `super().complete` and bypasses the custom `complete` span; unify instrumentation before trusting current counters.

**What remains unknown:** current traffic/audio length, effective provider model/region, token usage, actual invoice, deployed gateway controls, external report update semantics and current test results. Savings percentages and implementation dates should be based on those measurements, not invented from source size.
