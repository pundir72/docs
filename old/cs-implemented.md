# Implemented Security, Reliability and Cost Optimizations

This guide explains what was changed, how it works at runtime, and where the
implementation can be shown in the code. Paths are relative to the two project
repositories:

- Backend: `customer-agent/DataHandling/`
- Frontend: `customer-agent-frontend/`

## Short meeting overview

> I secured patient access, corrected form ownership and attempt identity,
> strengthened the WebSocket and upload boundaries, made report jobs recoverable,
> reduced unnecessary AI calls, centralized AI models and telemetry, removed
> privacy and TLS weaknesses, repaired the frontend build and dependencies, and
> added regression coverage and deployment documentation. These changes are
> implemented and verified locally. Production secrets, trusted token issuance,
> monitoring dashboards and target-environment integration tests remain deployment
> responsibilities.

## Main runtime path after these changes

```text
Patient link with signed token
        |
        v
Frontend consumes token and opens REST/WebSocket connection
        |
        v
Backend verifies signature + expiry + scope + patient identity
        |
        v
Interview answer is given a requestId and processed once
        |
        v
LangGraph updates the exact userId + formId + attemptId record
        |
        v
MongoDB saves draft/in_progress/completed status
        |
        v
Updated question, progress or completion response returns to the frontend
```

---

## 1. Secure authentication for patient APIs and WebSocket sessions

### How it works

- REST endpoints read the `Authorization: Bearer <token>` header.
- The WebSocket receives the token in its first `start_interview` message.
- The backend validates the token before looking up the patient.
- Missing, invalid or unauthorized credentials fail closed.
- Every patient-data REST route calls the common authentication boundary.

### Where it is implemented

- Token verification and authorization: `app/security/access_tokens.py:93`
- HTTP authentication helpers: `server.py:203`
- WebSocket endpoint and early authentication: `server.py:1988`
- Frontend REST token attachment: `customer-agent-frontend/src/config/auth.ts:34`
- Frontend WebSocket token attachment: `customer-agent-frontend/src/hooks/useWebSocket.ts:401`

### Verification

- `tests/test_access_tokens.py`
- `tests/test_access_boundary.py`

## 2. Short-lived, patient-specific access links

### How it works

The token is a strict HS256 JWT containing issuer, audience, subject, role,
scopes, issued-at time and expiry. A patient token is valid only when its subject
matches the patient being accessed. The default maximum lifetime is four hours
(`14,400` seconds), and deployment can configure a shorter value.

The trusted invitation service creates the token. The frontend only consumes it
from `#access_token=...`, removes it from the visible URL and stores it for the
browser session. The application intentionally exposes no public token-creation
API.

### Where it is implemented

- Strict JWT creation/validation contract: `app/security/access_tokens.py:93`
- Patient authorization: `app/security/access_tokens.py:223`
- Lifetime configuration: `app/config.py:33`
- Frontend fragment handling: `customer-agent-frontend/src/config/auth.ts:3`
- Local test-only generator: `scripts/generate_access_token.py`

### Production boundary

`AUTH_SIGNING_SECRET`, issuer and audience must be supplied through the deployment
secret manager, and the trusted clinician/invitation system must issue the links.

## 3. Prevention of cross-patient form access

### How it works

Authorization is based on both permission scope and patient binding:

- A patient can access only their own subject ID.
- A clinician can access only patient IDs listed in their token.
- An admin can access records only with the required operation scope.
- Directory access requires clinician/admin role plus `users:read`.

This check protects consent, form reads, progress, writes/uploads and WebSocket
operations. A patient ID in the URL alone is not trusted.

### Where it is implemented

- Role, scope and patient rules: `app/security/access_tokens.py:223`
- REST boundary: `server.py:230`
- WebSocket authorization calls: `server.py:2114`, `server.py:2834`, `server.py:3222`

### Verification

- Cross-user and role cases: `tests/test_access_tokens.py`
- Route/ordering contract: `tests/test_access_boundary.py`

## 4. Exact-record database update and deletion

### Original problem

An empty-form cleanup could delete using `formId` alone. Because many patients
use `FRM-01`, that filter could target another patient's form.

### How it works now

Before a mutation, `build_owned_form_filter()` confirms the loaded document belongs
to the expected patient and form. It produces a filter containing the exact MongoDB
`_id`, stored `userId` and `formId`. The delete therefore targets one verified
document, not every document sharing a form name.

### Where it is implemented

- Exact ownership filter: `app/db/ownership.py:11`
- Protected placeholder deletion: `server.py:2974`
- Form-scoped query helpers: `app/db/queries.py`

### Verification

- `tests/test_db_ownership.py`
- `tests/test_db_queries.py`

## 5. Separate attempt IDs for repeat assessments

### How it works

- `formId` identifies the questionnaire, for example `FRM-01`.
- `attemptId` identifies one patient's individual completion.
- A new assessment receives an opaque `ATT-<UUID>` value.
- Resume uses the existing attempt ID.
- Reads, writes, uploads and frontend URLs carry the attempt ID.
- A unique MongoDB index covers `(userId, formId, attemptId)`.
- Legacy records without `attemptId` remain readable, but their writes cannot
  accidentally match a newer attempt.

### Where it is implemented

- Attempt creation, query scope and index: `app/forms/attempts.py`
- Persistence integration: `server.py:1219`
- WebSocket propagation: `server.py:2169`
- Frontend URL/state: `customer-agent-frontend/src/pages/Index.tsx`
- Frontend protocol: `customer-agent-frontend/src/hooks/useWebSocket.ts:401`

### Verification

- `tests/test_form_attempts.py`

## 6. Draft, in-progress and completed form states

### How it works

- An empty form is `draft` and receives an expiration date.
- A form with a meaningful answer becomes `in_progress` and loses draft expiry.
- A terminal interview becomes `completed` and receives `completedAt`.
- Completion is monotonic: an ordinary delayed save cannot return a completed form
  to draft or in-progress.

### Where it is implemented

- Lifecycle rules: `app/forms/lifecycle.py:39`
- Lifecycle persistence: `server.py:1338`
- Terminal save paths: `server.py:3807`, `server.py:3990`, `server.py:4064`

### Verification

- `tests/test_form_lifecycle.py`

## 7. Completed forms protected from draft cleanup

### How it works

The MongoDB TTL index now watches `expiresAt` and has a partial filter of
`status: draft`. In-progress and completed documents do not have `expiresAt`, so
MongoDB's abandoned-draft cleanup cannot remove them. Cleanup no longer depends on
a display title such as `New Form`.

### Where it is implemented

- TTL policy and index migration: `app/forms/lifecycle.py:99`
- Index initialization: MongoDB initialization in `server.py:557`

### Verification

- TTL/index and completion regression cases: `tests/test_form_lifecycle.py`

## 8. Duplicate WebSocket answers and reconnection handling

### How it works

- The frontend creates a unique `requestId` for each answer.
- Retries reuse that identity instead of relying on answer text.
- The backend keeps a bounded recent-request window per authenticated actor.
- The same request ID for the same question returns `submission_ack: duplicate`
  and is not processed twice.
- Reusing an ID for a different question is treated as a conflict.
- Two intentionally identical consecutive answers still work because their
  request IDs differ.

### Where it is implemented

- Backend idempotency window: `app/ws/idempotency.py`
- Backend message boundary: `server.py:3271`
- Frontend request ID generation: `customer-agent-frontend/src/components/TranscriptionInterface.tsx:1090`
- Frontend protocol/ack handling: `customer-agent-frontend/src/hooks/useWebSocket.ts`

### Verification

- `tests/test_ws_idempotency.py`

### Current scale boundary

The window is bounded and process-local. Multi-replica global idempotency would
require a shared store such as Redis or MongoDB.

## 9. Backend thread and frontend WebSocket resource leaks

### How it works

The obsolete `HealthAgent` keyword polling thread was removed. Previously, one
never-stopped daemon thread could be created for the global agent and for every
WebSocket agent even though it was not required by the active interview flow.

On the frontend, component unmount now:

- disables automatic reconnect;
- clears the reconnect timer;
- removes socket callbacks;
- closes connecting/open WebSockets once;
- clears audio buffers; and
- closes browser audio resources.

### Where it is implemented

- Backend cleanup: `src/llm/functionalities.py`
- Frontend socket teardown: `customer-agent-frontend/src/hooks/useWebSocket.ts:470`
- Frontend audio teardown: `customer-agent-frontend/src/hooks/useVoiceRecorder.ts:306`

### Verification

- Backend dead-audio/thread guards: `tests/test_legacy_audio_cleanup.py`
- Frontend TypeScript, ESLint and production build gates

## 10. Microphone audio duration and size limits

### How it works

- Each WebSocket frame is limited to 64 KB.
- A recording session defaults to a maximum of 10 MB and 180 seconds.
- Limits are checked during chunk ingestion and again at `audio_end`.
- The server clears the audio buffer after failure, completion or disconnect.
- Transcription runs only after the bounded recording is complete.

### Where it is implemented

- Configured limits: `app/config.py:57`
- Pure limit decision: `app/audio/limits.py`
- Audio start/end and chunk enforcement: `server.py:3261`, `server.py:4112`,
  `server.py:4249`

### Verification

- `tests/test_audio_limits.py`

## 11. PDF and image upload validation

### How it works

The complete batch is validated before any file is stored:

- bounded streaming read prevents unbounded memory use;
- maximum five files, 15 MB per file and 25 MB total by default;
- magic bytes must identify PDF, PNG, JPEG or WebP;
- extension and declared MIME must match detected content;
- empty and unsupported files are rejected;
- PDF page count is bounded;
- image dimensions and decoded pixels are bounded;
- image decoders verify content before AI processing; and
- Bedrock input limits are checked.

### Where it is implemented

- Limits: `app/config.py:79`
- Bounded reads: `app/uploads/reading.py`
- Signature, MIME, quota and complexity policy: `app/uploads/policy.py`
- Upload preflight: `server.py:4571`
- Document decoding checks: `docscanner/service.py`

### Verification

- `tests/test_upload_policy.py`

### Remaining production control

An approved malware/quarantine scanner and proxy-level request-body limit are
still required. File type validation is not a replacement for malware scanning.

## 12. Durable report processing with retries and recovery

### How it works

Uploaded reports no longer depend only on an in-memory background task:

1. Files pass validation and are stored in S3.
2. A `queued` job is inserted in MongoDB.
3. A worker atomically claims it with a lease.
4. Failed jobs move to `retry` with bounded exponential backoff.
5. Expired leases can be reclaimed after a worker crash/restart.
6. Maximum attempts prevent endless retries.
7. The worker renews leases during long operations.
8. Only the latest job may publish its result to the form, preventing an old
   upload from overwriting a newer report.
9. Worker startup and graceful shutdown are tied to FastAPI lifespan.

### Where it is implemented

- Job states, leases and retry policy: `app/jobs/reports.py`
- Worker lifecycle and result publication: `server.py:270`, `server.py:601`
- Upload admission and job creation: `server.py:4678`
- Configuration: `app/config.py:95`

### Verification

- `tests/test_report_jobs.py`

## 13. Fewer AI requests for titles and structured PROM answers

### Form titles

The old save path called Gemini again to create a short title. It now uses a pure,
deterministic title derived from the primary complaint, so normal saves avoid that
extra model request.

- Implementation: `app/forms/titles.py`
- Save integration: `server.py:1285`
- Tests: `tests/test_form_titles.py`

### Structured PROM answers

When the frontend submits valid structured answers, the backend parses and stores
them directly and deterministically advances to the next PROM question. It does
not call the general form-extraction model for that turn. Natural spoken answers
can still use AI mapping when needed.

- Parsing/advance logic: `app/forms/prom.py`
- Runtime bypass: `server.py:3317`, `server.py:3553`, `server.py:3744`
- Tests: `tests/test_prom_structured.py`

## 14. Central Gemini model configuration

### How it works

One immutable registry owns the models for general text, reasoning, audio,
grounded search and fallback rotation. All active provider call sites read from
this registry, so a model migration is reviewed in one location.

### Where it is implemented

- Registry: `app/ai/models.py`
- Environment contract: `.env.example`
- Active integrations: `server.py`, `app/audio/stt.py`, `src/llm/functionalities.py`

### Verification

- `tests/test_ai_models.py`

## 15. Retired and inconsistent model names removed

### How it works

Only reviewed stable IDs are accepted:

- `gemini-2.5-flash-lite`
- `gemini-2.5-flash`

Known retired IDs and unknown IDs raise an error during startup configuration
instead of failing later during a patient request. A `models/` prefix is safely
normalized.

### Where it is implemented

- Approved and retired sets: `app/ai/models.py:21`
- Fail-fast validation: `app/ai/models.py:52`

### Verification

- Registry and active-source scan: `tests/test_ai_models.py`

## 16. AI usage, latency and estimated-cost monitoring

### How it works

Every active Gemini, Google Speech and Bedrock invocation passes through one
telemetry boundary. It records bounded metadata only:

- provider, exact model, operation and status;
- request count and success/failure latency;
- text/audio/output/thinking and cached tokens when reported;
- Google Speech submitted audio seconds;
- estimated cost only when every used billing unit has a configured price; and
- pricing-table version.

Prometheus metrics are exposed through `/metrics`. Unknown billing contracts stay
explicitly unpriced instead of reporting a false cost.

### Where it is implemented

- Tracking, provider parsing and pricing: `app/observability/ai_usage.py`
- Metrics setup: `server.py:170`
- Provider call sites: `server.py`, `app/audio/stt.py`, `docscanner/client.py`,
  `src/llm/functionalities.py`, `src/llm/utils.py`

### Verification

- `tests/test_ai_usage.py`

### Remaining production work

The deployment must scrape `/metrics`, configure any contract-specific prices,
create alerts, aggregate cost per completed form and reconcile estimates against
provider invoices.

## 17. Patient information removed from normal logs

### How it works

- Routine logs record event type, counts, lengths, latency and error class rather
  than patient text, transcription content, form values, filenames or URLs.
- Patient/session identifiers are omitted by default or keyed-pseudonymized.
- Provider exception messages are reduced to exception class.
- Langfuse content capture defaults to off.
- Optional MCP PHI transfer defaults to off; a remote MCP endpoint also requires
  an authentication token.
- Raw transcription debug files are no longer written.

### Where it is implemented

- Privacy helpers: `app/observability/privacy.py`
- Active logging: `server.py`
- Langfuse gate: `server.py:396`
- MCP privacy/authentication gate: `app/mcp_client.py`
- Defaults: `.env.example`

### Verification

- `tests/test_observability_privacy.py`

### Governance boundary

Organizational retention, vendor agreements, access policy and data residency are
not code-only decisions and still need owner approval.

## 18. MongoDB TLS certificate validation

### How it works

All MongoDB construction paths use a shared verified client factory. PyMongo's
certificate and hostname verification remain enabled. URI options such as
`tlsAllowInvalidCertificates=true`, `tlsAllowInvalidHostnames=true` and
`tlsInsecure=true` are explicitly rejected. A private trusted CA can be supplied
through `MONGO_TLS_CA_FILE` without disabling validation.

### Where it is implemented

- Secure client factory: `app/db/connection.py`
- Active connector: `app/db/mongo.py`
- Server initialization: `server.py:557`
- Configuration: `app/config.py:20`

### Verification

- `tests/test_mongo_connection.py`

## 19. Frontend TypeScript and ESLint repairs

### How it was implemented

The frontend was corrected for missing imports, PROM metadata types, WebSocket
message types, Apollo handler compatibility, React hook ordering/dependencies,
browser API types, empty aliases, Tailwind module loading and generated shadcn UI
handling.

### Where to show it

- WebSocket types: `customer-agent-frontend/src/hooks/useWebSocket.ts`
- Browser/audio types: `customer-agent-frontend/src/hooks/useVoiceRecorder.ts`
- Environment types: `customer-agent-frontend/src/vite-env.d.ts`
- Lint configuration: `customer-agent-frontend/eslint.config.js`
- Build scripts: `customer-agent-frontend/package.json`

### Verification commands

```bash
npx tsc -b
npm run lint
npm run build
```

All three currently pass in the reviewed local frontend.

## 20. Vulnerable frontend dependencies updated

### How it was implemented

Direct and transitive packages were updated through reviewed lockfile changes,
including React Router, Vite, Lodash, PostCSS and affected dependency chains. The
application was then type-checked, linted and production-built to detect breaking
changes.

### Where it is implemented

- `customer-agent-frontend/package.json`
- `customer-agent-frontend/package-lock.json`

## 21. Both frontend npm audits report zero vulnerabilities

“Both audits” means two views of the same frontend dependency graph:

```bash
npm audit --omit=dev   # production/runtime dependencies only
npm audit              # complete runtime + development dependency graph
```

Both currently report zero known vulnerabilities. This is a point-in-time result;
the checks should run in CI because new advisories can be published later.

## 22. Backend image reduced from about 575 MB to 428 MB

### How it was implemented

- Replaced the single-stage image with a multi-stage build.
- Compilers/build tools exist only in the builder stage.
- Removed unused direct dependencies and pinned the runtime graph.
- Copied only required runtime source instead of the entire repository.
- Removed test, documentation, credential and deployment files from the image.
- Kept only required runtime packages such as ffmpeg, Poppler and `libgomp1`.
- Removed writable source-code overlays from Compose.

### Where it is implemented

- `deployment/Dockerfile`
- `requirements.in`
- `requirements-docker.txt`
- `.dockerignore`
- Root and deployment `docker-compose.yml`

### Verification

The current image builds and measures `428,272,512` bytes versus the recorded
approximately 575 MB baseline. Container contract coverage is in
`tests/test_container_contract.py`.

### Dependency-audit disclosure

The latest backend audit still identifies unpatched advisories in Chroma server
APIs and a transitive NLTK model-path API. The application uses embedded Chroma,
does not expose the affected Chroma server API, and does not import the affected
NLTK APIs. These require formal risk acceptance or dependency removal/replacement,
plus upgrade when upstream fixes become available.

## 23. Generated files and unsafe repository artifacts removed

### How it was implemented

- Expanded `.gitignore` for secrets, keys, caches, coverage, logs, session files,
  temporary files and generated output.
- Expanded `.dockerignore` so these files cannot enter the container build context.
- Removed obsolete `update.sh` and `update-fast.sh` deployment scripts that used
  unsafe/destructive update behavior.
- Removed generated frontend `lint_output.txt`, obsolete `bun.lockb`, unused
  placeholder assets and committed development environment configuration.
- The Dockerfile uses an explicit runtime source allow-list.

### Where it is implemented

- Backend repository: `.gitignore`, `.dockerignore`, `deployment/Dockerfile`
- Removed scripts: `deployment/update.sh`, `deployment/update-fast.sh`
- Frontend repository: `.gitignore`, `.vercelignore`

This cleanup does not remove required runtime code or credentials from deployment;
credentials must be supplied securely at runtime.

## 24. Local development, API, deployment and architecture documentation

### Documentation added or corrected

- Root project and architecture overview: `../README.md`
- Backend documentation index: `docs/README.md`
- Local setup and safe testing: `docs/LOCAL_DEV.md`
- REST/WebSocket contract: `docs/PUBLIC_API.md`
- Deployment configuration and checks: `docs/DEPLOYMENT.md`
- Safe environment template: `.env.example`
- Runtime meeting guide: `docs/RUNTIME_FLOW_MEETING_GUIDE.md`
- Full optimization audit: `docs/CODEBASE_OPTIMIZATION_AUDIT.md`

### Verification

`tests/test_documentation_contract.py` checks important route, environment and file
contracts so known documentation drift does not silently return.

## 25. Regression tests added

The backend regression suite now covers the main high-risk boundaries:

| Area | Test files |
|---|---|
| Authentication/authorization | `test_access_tokens.py`, `test_access_boundary.py` |
| Patient/form ownership | `test_db_ownership.py`, `test_db_queries.py` |
| Attempt identity/lifecycle | `test_form_attempts.py`, `test_form_lifecycle.py` |
| Clinical escalation | `test_clinical_escalation.py`, `test_clinical_escalation_boundary.py` |
| Audio limits | `test_audio_limits.py` |
| Upload validation | `test_upload_policy.py` |
| Durable reports | `test_report_jobs.py` |
| WebSocket idempotency | `test_ws_idempotency.py` |
| AI registry/usage | `test_ai_models.py`, `test_ai_usage.py` |
| PROM integrity/direct answers | `test_prom_instruments.py`, `test_prom_instrument_boundary.py`, `test_prom_structured.py` |
| Async resource boundaries | `test_async_boundaries.py`, `test_blocking_runtime.py` |
| Privacy | `test_observability_privacy.py` |
| MongoDB TLS | `test_mongo_connection.py` |
| Container/dependencies | `test_container_contract.py` |
| Documentation | `test_documentation_contract.py` |
| Legacy/architecture guards | `test_legacy_audio_cleanup.py`, `test_graph_topology.py`, `test_functionalities_structure.py` |

The dependency-complete backend run passed `207/207` tests. The frontend passed
TypeScript, full ESLint and production build checks.

---

## Important production statement

Use this wording if asked whether everything is already live:

> The implementation and local verification are complete for the items described
> as fixed. They are not automatically production-deployed. Production readiness
> still requires deployment secrets, trusted access-link issuance, target MongoDB,
> S3, Bedrock and AI-provider integration tests, monitoring/alerts, malware scanning,
> CI gates and formal decisions for the documented third-party advisories and
> clinical/privacy policies.

This is more accurate than saying that local code changes alone make the entire
operational system production-ready.
