# Customer Agent — Implemented Changes and File Guide

This is a short, current-state list of the work implemented in the backend and
frontend development branches. It is grouped by feature so it can be explained
quickly during a developer review.

## 1. Backend orchestration and interview flow

| Files changed | What we changed |
|---|---|
| `DataHandling/server.py` | Hardened the main REST/WebSocket flow, isolated each connection's agent state, added safe start/resume/new-attempt handling, moved blocking work away from the async event loop, improved persistence, upload handling, clinical escalation, duplicate-request handling, and prompt-leak barriers. |
| `DataHandling/src/graph/graph.py` | Simplified the primary LangGraph topology and removed unnecessary sequential processing. |
| `DataHandling/src/graph/edges.py` | Corrected routing between extraction, validation, uploads, questions, summaries, and corrections. |
| `DataHandling/src/graph/state.py` | Cleaned the shared interview-state contract and removed obsolete correction-routing state. |
| `DataHandling/src/graph/server_adapter.py` | Kept graph results, progress, attachments, form identity, and WebSocket responses synchronized. |
| `DataHandling/src/graph/nodes/extract.py` | Combined extraction and intent work, reduced avoidable AI latency, and removed the incorrect normal-answer correction branch. |
| `DataHandling/src/graph/nodes/first_turn.py` | Improved first-message extraction and routing. |
| `DataHandling/src/graph/nodes/generate.py` | Improved next-question/summary transitions and deterministic referral handling. |
| `DataHandling/src/graph/nodes/correction.py` and `nodes/summary.py` | Restricted correction processing to the summary-confirmation flow and improved summary response handling. |
| `DataHandling/src/graph/pure_functions/form_extraction.py` | Improved structured field extraction, referral extraction, negative answers, corrections, and PROM-safe behavior. |
| `DataHandling/src/graph/pure_functions/intent_detection.py` | Removed broad keywords such as ordinary “sorry/actually/change” usage from active correction routing, preventing repeated correction messages during normal intake. |
| `DataHandling/src/graph/pure_functions/reasoning_extractor.py` and `summary.py` | Improved extraction/summary behavior and reduced unsafe or duplicate processing. |
| `DataHandling/src/llm/functionalities.py` and `src/llm/utils.py` | Reduced the legacy `HealthAgent` implementation, retained it as a fallback, and aligned provider/model handling with the active graph. |

## 2. Form data, ownership and repeat assessments

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/db/ownership.py` | Added reusable patient/form ownership checks so one patient cannot load another patient's form. |
| `DataHandling/app/db/queries.py` | Centralized ObjectId/string-compatible, user-scoped MongoDB queries and prevented broad form updates/deletes. |
| `DataHandling/app/db/connection.py` and `app/db/mongo.py` | Centralized MongoDB access and enforced TLS certificate/hostname verification. |
| `DataHandling/app/forms/attempts.py` | Added opaque `attemptId` values and indexes so repeated assessments remain separate. |
| `DataHandling/app/forms/lifecycle.py` | Added monotonic `draft`, `in_progress`, and `completed` lifecycle rules; only drafts can expire. |
| `DataHandling/app/forms/progress.py` | Centralized form and section completion calculation. |
| `DataHandling/app/forms/titles.py` | Replaced an unnecessary AI title request with deterministic form titles. |
| `DataHandling/app/forms/instruments.py` and `app/forms/prom.py` | Added stable PROM question identity, administered-question snapshots, structured-answer validation, and safe scoring state. |

## 3. WebSocket, frontend and audio reliability

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/ws/idempotency.py` | Added bounded request-ID tracking so retries do not save or process the same answer twice. |
| `DataHandling/app/runtime/blocking.py` | Added a bounded thread/executor boundary for synchronous database and provider work. |
| `DataHandling/app/audio/limits.py` | Added server-side recording duration, chunk, and total-byte limits. |
| `DataHandling/app/audio/stt.py` | Centralized the audio model, improved fallback handling, moved the Gemini instruction to `system_instruction`, and rejects prompt-contaminated transcripts before returning them. |
| `DataHandling/app/content_safety.py` | Added deterministic detection and recursive removal of leaked internal transcription instructions. |
| `../customer-agent-frontend/src/hooks/useWebSocket.ts` | Prevented parallel sockets, cleaned timers/buffers on unmount, stopped unrecoverable reconnect loops, handled duplicate acknowledgements, and aligned messages with the backend contract. |
| `../customer-agent-frontend/src/hooks/useVoiceRecorder.ts` | Improved MediaRecorder/audio cleanup and bounded recording behavior. |
| `../customer-agent-frontend/src/components/TranscriptionInterface.tsx` | Prevented auto-send/manual-send duplication, added request IDs, improved voice state cleanup, consent checks, upload behavior, and structured PROM controls. |

## 4. Clinical safety, privacy and observability

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/clinical/escalation.py` | Added deterministic detection and handling for urgent clinical risk, with a dedicated stop-interview WebSocket response. |
| `DataHandling/app/observability/privacy.py` | Added pseudonymous identifiers and safe exception-type logging to avoid normal logs containing patient answers or credentials. |
| `DataHandling/app/observability/ai_usage.py` | Added provider/model/operation call counts, latency, token/audio usage, failures, and estimated-cost metrics. |
| `DataHandling/app/ai/models.py` | Centralized approved Gemini models, defaults and fallbacks; retired or unapproved model IDs now fail configuration validation. |
| `DataHandling/app/mcp_client.py` | Made the optional MCP integration safer and kept it out of the required production path. |

## 5. Upload and report-processing reliability

| Files changed/added | What we changed |
|---|---|
| `DataHandling/app/uploads/policy.py` | Added file-count, per-file, total-size, signature/MIME, PDF-page, image-pixel, and image-dimension validation. |
| `DataHandling/app/uploads/reading.py` | Added bounded streaming reads so an oversized upload is rejected before consuming unlimited memory. |
| `DataHandling/upload/s3_client.py` | Improved safe S3 upload/download behavior and metadata handling. |
| `DataHandling/app/jobs/reports.py` | Replaced fragile background-only processing with MongoDB-backed jobs, leases, retries, recovery, and stale-result protection. |
| `DataHandling/docscanner/service.py` | Added safe PDF/image inspection, bounded rendering, multi-report limits, and stricter model-output handling. |
| `DataHandling/docscanner/client.py` | Centralized Bedrock configuration, retries and usage monitoring. |

## 6. Frontend configuration, quality and dependency work

| Files changed | What we changed |
|---|---|
| `../customer-agent-frontend/src/config/api.ts`, `config/policy.ts`, `utils/api-config.ts` | Centralized API, WebSocket, consent and environment URL validation; added controlled HTTP/IP support for isolated UAT. |
| `../customer-agent-frontend/vite.config.ts` and `src/vite-env.d.ts` | Added required build-time configuration validation and TypeScript environment definitions. |
| `../customer-agent-frontend/src/App.tsx`, `pages/Index.tsx`, `pages/ConsentPage.tsx`, `pages/FormSelection.tsx` | Improved patient/form URL routing, consent redirect behavior and form selection. |
| `../customer-agent-frontend/src/utils/graphql-client.ts` and data hooks | Removed unsafe defaults and aligned configured GraphQL requests. |
| `../customer-agent-frontend/eslint.config.js` and affected components | Fixed TypeScript/ESLint issues and unsafe loose types. |
| `../customer-agent-frontend/package.json` and `package-lock.json` | Updated vulnerable dependencies and standardized npm lockfile usage. |
| `../customer-agent-frontend/.gitignore`, `.dockerignore`, `.vercelignore` | Prevented local environment files, build output and unrelated artifacts from entering deployments/repository history. |

## 7. Deployment and repository cleanup

| Files changed/added | What we changed |
|---|---|
| `DataHandling/deployment/Dockerfile` | Changed to a multi-stage, allow-list-based runtime image and removed tests, credentials, local data and build tools from the final image. |
| `docker-compose.dev-isolated.yml` | Added an isolated backend service/container, port, volumes and network so UAT does not replace the existing production container. |
| `../customer-agent-frontend/Dockerfile.dev`, `docker-compose.dev-isolated.yml`, `nginx.dev.conf` | Added an isolated production-style frontend build served by Nginx on its own UAT port. |
| `DataHandling/.env.example` and frontend `.env.example` | Documented required configuration names without committing real secrets. |
| Root/DataHandling `.gitignore` and `.dockerignore` | Excluded credentials, generated audio/transcripts, reports, caches, local databases and build artifacts. |
| `DataHandling/deployment/update.sh`, `update-fast.sh`, the temporary token generator, stale output files and unused frontend assets | Removed unsafe, generated or obsolete repository/deployment artifacts. |
| `DataHandling/requirements.in`, `requirements-docker.txt`, `requirements.txt` | Separated direct dependencies and pinned the reproducible container dependency set. |

## 8. Documentation and tests

| Files changed/added | What we changed |
|---|---|
| `README.md` and `DataHandling/docs/README.md` | Replaced outdated architecture descriptions with the current application boundaries. |
| `DataHandling/docs/PUBLIC_API.md` | Documented current REST/WebSocket messages, form identity and persistence behavior. |
| `DataHandling/docs/LOCAL_DEV.md` | Added safe local configuration, startup and verification steps. |
| `DataHandling/docs/DEPLOYMENT.md` and `DEV_ISOLATED_DEPLOYMENT.md` | Added deployment checks and isolated shared-server UAT instructions. |
| `DataHandling/tests/test_*.py` | Added regression coverage for ownership, query scope, attempts, lifecycle, PROMs, uploads, report jobs, WebSockets, async boundaries, clinical escalation, privacy, model configuration, transcription prompt leakage and interview correction routing. |
| Frontend build/lint/audit workflow | Verified TypeScript compilation, ESLint, production build and npm dependency audit after the frontend changes. |

## 9. Important current-state clarifications

- A separate customer-agent access-token system was initially implemented, but
  it was intentionally removed after the product owner confirmed that the
  existing OTP/consent integration must remain the active flow. Do not present
  patient-link tokens as a current feature.
- Direct public HTTP/IP support was added only for isolated development/UAT. It
  is not the recommended production exposure model; production should use HTTPS,
  WSS and the approved upstream boundary.
- `context-layer/` was added by the earlier developer and later merged into this
  branch. We reviewed and documented how it connects, but it should not be
  presented as part of our optimization implementation.

## 10. Short meeting summary

> We hardened the complete patient-intake path across frontend, WebSocket,
> LangGraph, MongoDB and external AI services. The main results are isolated
> patient sessions, correct form ownership and repeat-attempt storage, fewer and
> safer AI calls, reliable audio/report processing, privacy-safe monitoring,
> prompt-leak and clinical-safety protection, cleaned deployment artifacts, and
> regression coverage for the critical flows. We also added isolated backend and
> frontend Docker deployment files for UAT without replacing production.

