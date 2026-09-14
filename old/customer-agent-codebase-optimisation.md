# Customer Agent Codebase Optimization Audit

Audit date: 25 August 2026  
Scope: customer-agent backend and customer-agent-frontend  
Purpose: identify production risks, legacy code, unnecessary work, cost drivers, and a safe optimization order before changing behavior.

## 1. Executive Summary

The application works as an AI-assisted patient intake system, but its current implementation is more expensive and harder to operate than necessary. The main cost is repeated Gemini processing on every interview turn, followed by an oversized backend dependency/container footprint and resource leaks in long-running sessions.

The first optimization release should not begin with a large rewrite. It should address five high-value areas:

1. Remove unnecessary AI calls from PROM answers and form-title generation.
2. add accurate per-feature token, latency, and cost telemetry;
3. stop per-WebSocket thread leaks and track background tasks;
4. correct authentication, secret handling, MongoDB TLS, and PROM persistence risks;
5. slim and pin backend dependencies, then split the large server module incrementally.

A normal substantive intake answer can currently cause approximately three model calls: form extraction, next-question generation, and title generation during persistence. Some paths add classification, correction, summary, personalization, voice mapping, or report-processing calls. Full conversation history is repeatedly included, so input-token cost grows throughout the interview.

No reliable percentage saving should be promised until production usage is measured. However, the first two AI-call changes are directly countable:

- structured PROM answers can avoid one Gemini 2.5 Flash call per answer;
- deterministic/cached titles can avoid approximately one Flash-Lite call per normal saved turn.

Together, these should materially reduce AI requests without changing the clinical conversation.

## 2. Current Technical Baseline

### Backend

- Python/FastAPI service with HTTP and WebSocket endpoints.
- LangGraph is the primary interview orchestration path.
- `HealthAgent` remains as the shared LLM wrapper and legacy fallback implementation.
- MongoDB stores users, forms, tagged PROM questions, consent state, appointments, and form progress.
- Gemini 2.5 Flash performs reasoning-grade extraction and audio transcription.
- Gemini 2.5 Flash-Lite is the primary general text model.
- Google Cloud Speech-to-Text is the audio fallback.
- ChromaDB supports optional Stance/medical retrieval.
- AWS S3 stores uploaded reports; AWS Bedrock processes report images/PDF pages.
- Optional Langfuse and Prometheus instrumentation is present.

Measured local baseline:

- 13,581 Python lines across 34 application/test files included in the count;
- 2,270 comment-only lines;
- `server.py`: approximately 4,670 lines;
- `src/llm/functionalities.py`: approximately 3,908 lines;
- local backend Docker image: 575,028,684 bytes, approximately 548 MiB / 575 MB;
- Docker build observed at roughly seven minutes because dependency resolution backtracked and installed a large transitive graph;
- one smoke-test script, dependent on a running service; no meaningful unit or isolated integration test suite.

### Frontend

- React 18, TypeScript, Vite, React Router, TanStack Query, shadcn/Radix UI.
- WebSocket carries the live interview protocol.
- GraphQL loads patients, appointments, and centers from the main Stance API.
- REST endpoints load forms, consent, progress, and attachments from customer-agent.

Measured local baseline:

- 9,665 TypeScript/TSX lines across 83 source files;
- `TranscriptionInterface.tsx`: approximately 2,109 lines;
- `useWebSocket.ts`: approximately 431 lines;
- production bundle: approximately 380 KB JavaScript, 119 KB gzip;
- ESLint: 52 errors and 10 warnings;
- TypeScript check: 8 errors;
- Vite production build passes because it does not perform a complete TypeScript check.

## 3. Critical Correctness and Security Issues

These should be fixed before aggressive performance refactoring.

### 3.1 Backend routes and WebSocket sessions are not authenticated

The backend accepts a user ID supplied by the client and verifies only that the user exists. It does not verify that the caller is authorized to access that user. The REST endpoints similarly expose user/form/consent/attachment operations without an authenticated identity and ownership check.

Impact:

- a known or discovered user ID may allow unauthorized reading or modification of patient data;
- WebSocket and attachment traffic is not protected by the configured but unused `WS_TOKEN`;
- cost controls can be bypassed by unauthenticated callers.

Required change:

- validate a signed access token on REST and WebSocket connections;
- derive identity/permissions server-side;
- authorize every form operation against the patient/clinician relationship;
- add per-identity and per-IP rate limits backed by a shared store when running more than one process.

### 3.2 A frontend API credential is public and has a hardcoded fallback

`VITE_API_KEY` is embedded into browser JavaScript by design, and a fallback credential is committed in `src/utils/api-config.ts`. A Vite variable cannot be treated as a secret.

Required change:

- rotate the exposed credential;
- remove the fallback from source;
- replace it with short-lived user/session authorization or a server-side proxy where a service credential is required.

### 3.3 MongoDB certificate validation is disabled

The primary Mongo client uses `tlsAllowInvalidCertificates=True`.

Impact: the application can accept an invalid TLS certificate, weakening protection for patient records.

Required change: use the correct CA chain and normal certificate validation. Do not solve certificate failures by bypassing validation.

### 3.4 Patient health information is written to application logs and observability traces

Current logging includes user text, transcriptions, extracted answers, PROM answers, identifiers, and form details. Langfuse tracing can also receive prompts and outputs.

Required change:

- define a PHI logging policy;
- redact or hash identifiers;
- remove raw patient text from routine logs;
- configure limited retention and access controls;
- confirm vendor agreements and data residency before sending clinical data to external observability or AI providers.

### 3.5 PROM documents can be treated as abandoned forms

The MongoDB TTL index removes documents whose title remains `New Form` after seven days. PROM forms do not populate the normal `Present Complaint`, so completed PROM documents can retain that title.

Required change: base expiration on an explicit `status: abandoned` and `expiresAt` field, never on a display title.

### 3.6 PROM clinical wording is changed dynamically

PROM personalization can replace standardized time windows such as “past four weeks” with the patient's complaint duration. This may invalidate a standardized questionnaire and makes scores non-comparable. Personalized question text is also used as a MongoDB field key, creating unstable schemas.

Required change:

- keep validated instrument wording and options immutable;
- store answers by stable question ID;
- keep display text and instrument version as metadata;
- add instrument scoring and interpretation rules if the product relies on PROM results.

### 3.7 Known runtime defects

- `if users_collection` evaluates a PyMongo `Collection` as a boolean and can raise on the invalid-user error path.
- `fetch_tagged_questions` does not return a consistent three-value tuple on every early-return path.
- tagged-question fallback can search by user without sufficiently constraining the form, which can select the wrong questionnaire.
- `DirectFormPage` is imported by the frontend but does not exist.
- frontend `QuestionMeta.question_scales` is defined inconsistently as `string[]` and `string[][]`.
- the WebSocket message union does not contain the handled `token` message shape.
- Apollo error handler types no longer match the installed Apollo version.

## 4. Primary Cost Drivers

### 4.1 Repeated AI calls per interview turn

Typical normal turn:

1. Gemini 2.5 Flash extracts structured form data.
2. Flash-Lite generates the next question.
3. A background save calls Flash-Lite again to generate a form title.

Conditional additional calls include first-turn visit classification, report-intent classification, correction detection/application, summary generation, off-topic classification/answering, PROM personalization, voice answer mapping, and document summarization.

The title is generated again on ordinary saves whenever a primary complaint exists. A display title does not need an LLM and should never be regenerated on every turn.

Optimization:

- create a deterministic title from a normalized complaint, or generate it once and retain it;
- only update it when the complaint materially changes;
- do not run title work in the persistence layer.

### 4.2 PROM answers run unnecessary graph extraction

The server already maps typed/button PROM responses directly to stable question IDs before invoking LangGraph. LangGraph then runs reasoning extraction anyway, and the server later overrides the generated response with the next PROM batch.

Optimization: implement a dedicated PROM state machine that validates and persists structured answers without general intake extraction/question generation. Use an LLM only for genuinely free-form multi-answer voice mapping.

Expected direct result: one Gemini 2.5 Flash request removed per structured PROM answer.

### 4.3 Prompt size grows throughout the interview

Extraction and question-generation prompts repeatedly include broad form and conversation context. As history grows, later turns cost more and take longer than early turns.

Optimization:

- send the latest answer, current section, missing fields, compact structured patient facts, and a short rolling summary;
- do not resend complete raw history unless correction handling requires it;
- use stable prompt prefixes/context caching only after measuring whether the retained prefix is large and reusable enough to justify caching storage cost.

### 4.4 Model choice is not task-specific enough

Gemini 2.5 Flash is used for all reasoning extraction and audio transcription. Several extraction tasks may be suitable for Flash-Lite or deterministic parsing, but this requires accuracy evaluation against representative conversations.

Optimization:

- build a small de-identified evaluation dataset;
- compare Flash, Flash-Lite, and deterministic extraction field by field;
- use the cheapest model that meets the acceptance threshold per task;
- keep a more capable model only for ambiguous correction and clinical extraction cases.

### 4.5 Audio transcription has two paid providers in a fallback chain

Gemini audio transcription is attempted first; Google Cloud Speech-to-Text is used after failure or unusable output. This is resilient, but failed Gemini attempts can add latency and possibly billable usage before Google fallback.

Optimization:

- measure word error rate, medical-term accuracy, latency, failure rate, and cost per minute for both providers;
- choose one primary based on evidence;
- retain fallback only for clear technical failures;
- enforce maximum audio duration and file size before provider calls;
- do not claim the health endpoint's fixed `google-cloud-speech` label represents the actual primary path.

### 4.6 Report processing is unbounded background work

Each uploaded report can cause S3 operations, PDF rasterization, and Bedrock requests. The task is created inside the web process without a durable queue, concurrency cap, retry policy, idempotency key, or cost limit.

Optimization:

- validate page/file limits before upload;
- move OCR/summarization to a bounded worker queue;
- deduplicate by content hash;
- store job state and retries;
- summarize only clinically relevant pages where product requirements allow it.

### 4.7 Google Search grounding path uses a shut-down model

`_answer_with_web_search` calls `gemini-2.0-flash`. Official Google documentation lists Gemini 2.0 Flash as shut down on 1 June 2026. The function appears unused now, but leaving it creates a broken legacy path and possible failed-request latency if reconnected.

The model fallback list also includes `gemini-2.0-flash` and `gemini-1.5-flash`. These should not be treated as operational fallbacks. Stable Gemini 2.5 Flash and Flash-Lite currently have no announced shutdown date, but model IDs should remain configuration rather than source constants.

## 5. Resource and Infrastructure Waste

### 5.1 A daemon thread leaks for every WebSocket connection

Each connection constructs a `HealthAgent`. Its constructor starts a keyword-checking daemon thread that loops until `stop_flag` becomes true. The WebSocket cleanup never sets that flag, and the thread's bound method retains the agent.

Impact:

- disconnected sessions leave threads and agent objects alive;
- memory and scheduler overhead grow with historical connections;
- the application eventually needs restarts or a larger instance.

Required change: remove the obsolete keyword thread entirely if relevancy actions are unused. Otherwise provide explicit lifecycle cancellation in WebSocket `finally` and verify thread count under repeated connect/disconnect load.

### 5.2 Fire-and-forget work is not tracked

Mongo saves and OCR tasks use `asyncio.create_task`/`to_thread` without a task registry, durable queue, or shutdown drain. Concurrent saves can complete out of order and overwrite newer form state. A process restart can drop work after the response was sent.

Required change:

- serialize saves per form with a monotonically increasing version;
- use atomic field updates where possible;
- retain and drain critical task handles during shutdown;
- use a queue for durable report work.

### 5.3 Backend image and dependency graph are oversized

The 575 MB image installs build tools and a broad Python dependency tree in the runtime image. `llama-index` is installed as a meta-package alongside specific LlamaIndex packages, pulling unrelated integrations such as OpenAI, Llama Cloud, data libraries, Kubernetes-related packages, tokenizers, and ONNX runtime. Broad version ranges caused resolver backtracking and non-reproducible installs.

Optimization:

- remove the `llama-index` meta-package and depend only on imported subpackages, or replace the remaining wrapper with the already-used direct Google GenAI SDK;
- remove confirmed-dead packages such as `edge-tts` and, after deleting dead helpers, `numpy` if no active use remains;
- decide whether Chroma retrieval is an actual production feature; retain and test its exact subpackages or remove its code/data/dependencies together;
- pin every direct production dependency and use a lock/constraints file;
- use a multi-stage build so compilers and headers do not remain in runtime;
- remove `git` from runtime unless runtime code genuinely executes Git;
- use Python rather than `curl` for the health check if `curl` has no other purpose.

### 5.4 Production runs from a read-write source bind mount

Compose copies code into the image and then overlays `/app` with `./DataHandling:/app:rw`. The image is therefore not the actual immutable production artifact. Host leftovers can remain active, and rollback/reproduction is unreliable.

Required change: production should run an immutable versioned image with only required data/credential mounts. Keep source bind mounts in a development-only Compose override.

### 5.5 Deployment script can produce partial/stale deployments

`update-fast.sh`:

- syncs production and development from the same local source;
- restarts without rebuilding after dependency changes;
- does not use `rsync --delete`, so removed source files remain remotely;
- uses `eval` for the rsync command;
- contains a fixed host, user, directory, and key filename;
- clears only selected bytecode directories.

Required change: replace this with a versioned build/deploy pipeline, environment-specific promotion, health validation, and rollback. At minimum, detect dependency changes and force a rebuild.

## 6. Legacy and Unwanted Code

### Backend

- approximately the first 400 lines of `server.py` are an older commented server implementation;
- `functionalities.py` contains a large commented duplicate implementation near its end;
- `HealthAgent` remains a full alternate interview engine although LangGraph is primary;
- the LangGraph `classify_intent` node is a documented no-op but still exists in graph topology and UI thought labels;
- `tagged_cleanup_done` remains in state but is unused;
- `save_audio` and `text_to_speech` are not called by the active server path;
- active TTS references `gtts`, which is not imported/installed, while unrelated `edge-tts` remains installed;
- `_answer_with_web_search` is unused and references a shut-down model;
- two MongoDB initialization implementations exist (`server.py` and `app/db/mongo.py`), with different TLS behavior;
- configuration, serializers, and some service code were extracted, but the active server still contains most routing, persistence, domain, and orchestration logic.

### Frontend

- `DirectFormPage` is a stale import for a missing file;
- `ConsentPage` exists while routing redirects to the external consent site;
- `VITE_MODE` is declared but unused;
- production/development backend addresses are partly hardcoded in source;
- `TranscriptionInterface.tsx` combines protocol handling, clinical flow state, audio, rendering, uploads, timers, and notifications in one component;
- many shadcn/Radix components and application packages may be template leftovers; run an import-aware dependency audit before removal;
- duplicated/incompatible WebSocket protocol types have already produced TypeScript errors.

Legacy removal must be performed after characterization tests are added. Commented code should be deleted from active files and recovered from Git history when needed.

## 7. Observability Problems

Prometheus counters for LLM calls, latency, fallback rotation, and interview turns are declared but not incremented in the active call wrapper. Therefore `/metrics` does not currently provide the information needed to validate optimization.

The logged cost calculation is also incorrect:

- it applies one fixed price to every model;
- it estimates input at $0.15/M and output at $0.60/M;
- current standard paid pricing differs by model and modality;
- audio and grounded-search charges are not captured;
- direct Google GenAI calls outside `HealthAgent.llm_complete` bypass this accounting.

Required telemetry dimensions:

- feature: extraction, question, title, PROM mapping, summary, STT, report OCR, off-topic/RAG;
- provider and exact model;
- input/output/audio tokens or billed seconds/pages;
- latency, success, fallback, retry, and cached status;
- anonymized session/form type;
- estimated cost using a versioned pricing table;
- total AI calls and cost per completed intake/PROM.

Do not use patient ID as a high-cardinality Prometheus label.

## 8. Frontend Quality and Performance

The bundle size is acceptable for the current application, so frontend bundle reduction is not the first cost priority. Correctness and maintainability are more urgent.

Current checks:

- ESLint fails with 52 errors and 10 warnings;
- TypeScript fails with 8 errors;
- many hook-rule failures arise because React component functions are named with a leading underscore and are no longer recognized as components by the lint rule;
- several explicit `any` values hide protocol incompatibilities;
- hook dependency warnings risk stale closures;
- no frontend test script or typecheck script exists.

Optimization:

- make `npm run check` run TypeScript, ESLint, tests, and build;
- define the WebSocket protocol once with discriminated union types and runtime validation;
- split `TranscriptionInterface` into a session controller plus focused UI components/hooks;
- lazy-load report upload and other secondary screens only if bundle analysis shows value;
- remove unused UI libraries/components only after import analysis.

## 9. Testing Gaps

The current backend smoke script only validates a live deployment and cannot safely exercise the production-connected database. There are no regression tests for the most sensitive behavior.

Minimum test foundation before refactoring:

1. form loader and field validation unit tests;
2. graph routing tests with a fake LLM;
3. PROM mapping, persistence, resume, scoring, and TTL tests;
4. Mongo ID normalization and authorization tests;
5. concurrent save ordering tests;
6. WebSocket protocol and reconnect tests;
7. repeated connection test proving thread/task cleanup;
8. audio provider fallback tests without live billing;
9. model-call budget tests asserting maximum calls per interaction;
10. de-identified golden-conversation extraction evaluations.

All automated tests must use isolated local/test collections and fake external providers by default.

## 10. Recommended Implementation Order

### Phase 0 — Establish measurements and safety gates

- add isolated test configuration and fake provider adapters;
- correct and activate AI usage metrics;
- record baseline calls, tokens, latency, errors, and completion rate by flow;
- add frontend typecheck/lint gates and backend tests to CI;
- rotate the browser-exposed credential.

### Phase 1 — Low-risk, high-return fixes

- stop/remove per-connection keyword threads;
- replace repeated LLM title generation with a deterministic/cached title;
- bypass LangGraph LLM extraction for structured PROM answers;
- remove shut-down model IDs from code/fallbacks;
- fix PyMongo boolean usage and tagged-question tuple/selection defects;
- correct PROM expiration logic;
- make frontend API/WS configuration consistent and remove the stale import;
- enforce input/audio/upload limits.

### Phase 2 — Data protection and reliability

- add token authentication and record-level authorization;
- restore MongoDB certificate verification;
- redact PHI logs and review third-party data handling;
- serialize/version Mongo saves;
- move OCR to a bounded durable worker;
- make production images immutable and deployments reproducible.

### Phase 3 — AI pipeline optimization

- use compact structured state instead of full conversation history;
- evaluate Flash-Lite for safe extraction tasks;
- merge extraction and next-question selection only if evaluations show equal quality;
- keep standardized PROM processing deterministic;
- benchmark and select the primary STT provider;
- cache only stable, sufficiently large shared prompt prefixes.

### Phase 4 — Structural cleanup

- split `server.py` into API routes, WebSocket protocol/session service, persistence repositories, AI providers, PROM engine, report jobs, and observability;
- retire the legacy `HealthAgent` interview engine after LangGraph parity tests pass;
- remove no-op graph nodes/state and commented duplicate implementations;
- reduce `functionalities.py` to narrowly scoped provider/domain services;
- split the frontend interview monolith and centralize protocol types;
- remove unused dependencies and build a minimal multi-stage image.

## 11. Cost-Saving Priority Matrix

| Change | Expected saving | Risk | Priority |
|---|---:|---:|---:|
| Skip reasoning extraction for structured PROM answers | One Flash call per structured answer | Low-medium | P0 |
| Stop generating title on every save | About one Flash-Lite call per normal saved turn | Low | P0 |
| Accurate per-feature usage metrics | Enables verified savings and budget alarms | Low | P0 |
| Stop WebSocket daemon-thread leak | Lower long-run memory/instance pressure | Low | P0 |
| Compact history/state in prompts | Large token reduction on later turns | Medium | P1 |
| Use cheaper model for evaluated extraction cases | Model-price reduction | Medium-high clinical risk | P1 |
| Bound/deduplicate report processing | Lower Bedrock/S3/CPU spikes | Medium | P1 |
| Choose one evidence-backed primary STT | Lower audio cost/fallback latency | Medium | P1 |
| Remove meta/unused dependencies and multi-stage build | Faster builds, smaller image, less storage/network | Low-medium | P1 |
| Retire duplicate legacy engine | Lower maintenance and regression cost | Medium-high | P2 |
| Frontend bundle trimming | Small current infrastructure saving | Low | P3 |

## 12. Pricing Reference Used for Planning

Pricing is external and must be rechecked before financial commitments.

As of the audit date, Google's standard paid Gemini API page lists:

- Gemini 2.5 Flash: $0.30 per million text/image/video input tokens, $1.00 per million audio input tokens, and $2.50 per million output tokens;
- Gemini 2.5 Flash-Lite: $0.10 per million text/image/video input tokens, $0.30 per million audio input tokens, and $0.40 per million output tokens;
- batch/flex token rates are lower but are generally unsuitable for interactive responses because the product requires immediate answers;
- Google Search grounding for Gemini 2.5 has a free allowance and then separate grounded-prompt charges.

Google Cloud Speech-to-Text pricing varies by API/version, logging choice, model, and volume. The code's synchronous fallback must be mapped to the actual billed SKU from the cloud bill before comparing providers.

Official references:

- Gemini API pricing: https://ai.google.dev/gemini-api/docs/pricing
- Gemini model deprecations: https://ai.google.dev/gemini-api/docs/deprecations
- Google Cloud Speech-to-Text pricing: https://cloud.google.com/speech-to-text/pricing

## 13. Acceptance Targets

Final targets should be set after a seven-day production baseline. Recommended initial engineering targets are:

- zero unauthenticated patient-data access;
- zero browser-embedded service secrets;
- zero leaked threads after WebSocket disconnect;
- zero LLM calls for typed/button PROM answers;
- no more than two AI text calls for a normal intake answer, with a longer-term target of one where quality allows;
- one title computation per form, normally deterministic;
- no raw PHI in ordinary logs;
- passing TypeScript, ESLint, backend tests, and production builds;
- pinned/reproducible dependency installation;
- materially smaller backend image and build time, measured after dependency cleanup;
- dashboarded AI cost per completed form, not merely cost per request.

## 14. Decision Required Before Implementation

The first implementation batch should be limited to behavior-preserving work:

1. telemetry correction;
2. WebSocket thread cleanup;
3. deterministic/cached titles;
4. structured PROM bypass;
5. runtime defect fixes;
6. frontend type/protocol fixes.

Security changes should begin in parallel at the design level because authentication must match the main Stance identity system. Large module splitting and legacy-engine deletion should wait until the above tests and baselines exist.
