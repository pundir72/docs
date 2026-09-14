# Stance Customer Agent — 20-Minute Project KT

## 1. Project goal

The Stance Customer Agent is a patient-facing AI intake application for
musculoskeletal (MSK) care. A patient opens a clinician-provided link, confirms
consent, and answers intake questions by voice, text, or structured controls.
The system converts the conversation into structured clinical form data,
supports report uploads, shows progress and a final summary, and saves the
result in MongoDB for clinical use.

The repository has two backend services:

- `DataHandling/`: the patient interview API, WebSocket, AI interview graph,
  speech transcription, form storage, and report processing.
- `context-layer/`: a separate assessment-test recommender. It reads a completed
  intake and recommends objective clinical tests when the dashboard calls it.

The React/Vite patient frontend is in the sibling repository
`../customer-agent-frontend/`.

## 2. Architecture in one view

```text
Clinician-provided patient link
        |
        v
React frontend --> consent-status REST API --> MongoDB consent record
        |
        +<====== WebSocket ======> FastAPI server
                                      |
                 typed answer --------+
                 voice --> Gemini STT --> Google Speech fallback
                                      |
                                      v
                              LangGraph interview
                         extract -> validate -> ask/summary
                                      |
                                      v
                              MongoDB customer-info

Report upload --> REST API --> validation --> S3 --> Mongo job --> Bedrock summary

Dashboard click --> context-layer --> intake + test catalogs --> recommendations
```

MongoDB is the persistent source of truth. WebSocket session state and duplicate
request tracking are process-local, while completed/partial forms and report jobs
are stored durably.

## 3. Main patient interview flow

1. **Link entry** — The patient opens `/{userId}/{formId}`. `FRM-01` is the
   standard intake; another form ID represents an assigned PROM/assessment.
2. **Consent check** — The frontend calls
   `GET /api/users/{userId}/consent`. If consent is absent, it opens the external
   consent application. When the patient returns, the frontend checks again.
3. **WebSocket connection** — The frontend connects to `/ws/{clientId}` and
   sends `start_interview` with `userId`, `formId`, and optional `attemptId`.
4. **Start or resume** — The backend validates that the user exists, creates a
   per-connection agent/session, and loads the matching MongoDB form when an
   attempt already exists.
5. **Patient answer**:
   - Typed answers are sent as `text_input`.
   - Voice is sent as `audio_start`, binary chunks, then `audio_end`.
   - The backend transcribes audio with Gemini; Google Cloud Speech is the
     independent fallback. The transcript is returned to the frontend, where the
     patient can edit it before/while it is auto-submitted after two seconds.
6. **Safety and duplicate checks** — The backend applies request-ID
   idempotency, audio limits, internal-prompt leak protection, and deterministic
   urgent clinical escalation checks before normal AI processing.
7. **LangGraph processing** — The current answer and saved state enter the graph:
   - extract structured answers into the form;
   - determine visit/report intent;
   - validate which required fields remain;
   - generate the next relevant question, request a report, or generate a summary.
8. **Response and save** — FastAPI sends `thought_update`, optional `token`, and
   the final `text_message` with progress. The updated form is saved to MongoDB.
9. **Completion/correction** — The patient reviews the summary. Confirmation
   completes the attempt; an explicit summary correction is detected, applied,
   and the updated summary is returned.

Primary graph path:

```text
welcome
  -> extract answer
  -> validate section
  -> ask next question / advance section / request report
  -> final summary
  -> confirm or correct
  -> complete
```

LangGraph is the primary runtime. `HealthAgent` in
`src/llm/functionalities.py` remains a fallback if graph initialization or graph
processing fails.

## 4. How questions are handled

### Standard intake (`FRM-01`)

`DataHandling/data/forms/FRM-01.md` is the editable form definition. It contains:

- the six sections and their fields;
- visit types and sections that can be skipped;
- plain-language field descriptions;
- predefined opening/grouped questions;
- the general-visit and referral questions.

`src/forms/loader.py` parses and caches this Markdown file. The graph does not
simply read a fixed question one by one. It uses the form definition, full chat
history, visit type, and missing fields to ask only the next relevant question.
`src/graph/nodes/generate.py` owns that decision. The referral question is
handled deterministically there so it is asked once.

### Assigned PROM forms (`FRM-02` and other non-default IDs)

`server.py::fetch_tagged_questions` loads the patient's assigned question list
from MongoDB `tagged-questions`, with support for embedded questions in
`customer-info`. ID-only entries are resolved against the `question-bank`
collection. The backend preserves stable question IDs, wording, types, options,
and instrument version, then sends `question_meta` so the frontend can render
text, choice, scale, rating, grid, date, number, upload, or multi-answer controls.

This application **consumes** question assignments; it has no question/category
admin UI and no create/edit/delete/publish question APIs. That administration is
owned by the upstream clinical/admin system and its MongoDB data.

## 5. Prompt and AI locations

| Purpose | Main file | What it controls |
|---|---|---|
| Welcome, first turn, next question, extraction, PROM and correction prompts | `DataHandling/src/prompts.py` | Main interview language and structured extraction rules |
| Form sections and editable intake wording | `DataHandling/data/forms/FRM-01.md` | FRM-01 fields, visit types and predefined questions |
| Dynamic next-question logic | `DataHandling/src/graph/nodes/generate.py` | Uses history and missing information; controls referral and summary transition |
| Answer extraction | `DataHandling/src/graph/nodes/extract.py` and `pure_functions/form_extraction.py` | Converts conversational answers into form fields |
| Speech-to-text instruction | `DataHandling/app/audio/stt.py` | Gemini audio transcription and Google Speech fallback |
| Report-analysis prompt | `DataHandling/docscanner/prompts.py` | JSON structure requested from Bedrock for uploaded reports |
| Recommender AI prompt | `context-layer/app/pipeline/llm_clinician.py` | Converts completed intake context into proposed assessment content |

Gemini model names are centrally validated in `DataHandling/app/ai/models.py`:

- General interview: `gemini-2.5-flash-lite` by default.
- Reasoning/form extraction: `gemini-2.5-flash` by default.
- Audio transcription: `gemini-2.5-flash` by default.
- Google-search model: `gemini-2.5-flash` by default.
- Speech fallback: Google Cloud Speech-to-Text v1.
- Report images/PDFs: Amazon Bedrock Nova, configured in
  `DataHandling/docscanner/client.py`.

## 6. API summary

FastAPI entry point: `DataHandling/server.py`.

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Backend and dependency/configuration health |
| `GET` | `/metrics` | Prometheus operational and AI-usage metrics |
| `GET` | `/api/users` | Search/list users |
| `GET` | `/api/users/{userId}/consent` | Read consent status |
| `POST` | `/api/users/{userId}/consent` | Record consent through this API |
| `GET` | `/api/users/{userId}/forms` | List the patient's intake attempts |
| `GET` | `/api/forms/{formId}?userId=...&attemptId=...` | Load a form attempt |
| `GET` | `/api/forms/{formId}/progress?userId=...` | Return section/progress state |
| `POST` | `/api/forms/{formId}/attachments` | Validate and upload reports |
| `WS` | `/ws/{clientId}` | Live interview, voice and text messages |

FastAPI also exposes `/docs`, `/redoc`, and `/openapi.json`.

Important WebSocket input types are `start_interview`, `start_new_form`,
`load_form`, `text_input`, `audio_start`, binary audio, `audio_end`, and
`end_session`. Important output types are `text_message`, `transcription`,
`thought_update`, `token`, `form_loaded`, `submission_ack`,
`clinical_escalation`, and `error`.

The patient application currently uses the existing external OTP/consent flow
and does not require a separate customer-agent Bearer token. Consent enables the
frontend interview, but it must not be described as independent record-level
authentication; production exposure depends on the approved upstream boundary.

## 7. Database and form lifecycle

Database names and collection configuration are loaded in
`DataHandling/app/config.py`; the connection is initialized in `server.py` using
TLS verification.

| Collection | Purpose |
|---|---|
| `users` | Patient/user lookup and possible profile consent state |
| `consentrecords` | Consent written by the external consent application |
| `customer-info` | FRM-01/PROM form data, status, progress, attempt and attachments |
| `tagged-questions` | Questions assigned to a patient/form |
| `question-bank` | Question text, type, options and instrument metadata |
| `customer-agent-report-jobs` | Durable report processing queue/retry state |
| `customer-agent-clinical-escalations` | Privacy-limited urgent-risk events |
| `appointments` | Appointment association when available |

`formId` identifies the questionnaire; `attemptId` identifies one submission of
that questionnaire. This prevents repeat assessments from overwriting older
attempts. Forms progress through `draft`, `in_progress`, and `completed`.
Only inactive drafts receive TTL expiry; completed forms are retained.

## 8. Report upload flow

1. The graph detects that the patient has reports and asks the frontend to show
   the upload control.
2. The frontend posts files to the attachments REST endpoint.
3. `app/uploads/` validates identity, size, count, MIME/signature, image
   dimensions, and PDF/page limits.
4. `upload/s3_client.py` stores accepted files in S3.
5. `app/jobs/reports.py` creates a Mongo-backed job with lease, retry, recovery,
   and latest-result protection.
6. `docscanner/service.py` safely renders PDFs/images and calls Bedrock through
   `docscanner/client.py`.
7. The report summary is written back into the correct form attempt.

## 9. Context-layer flow

The context-layer does not generate patient interview questions. It is a
separate service used after intake data exists:

```text
Dashboard request
 -> load applicable customer-info intake
 -> classify condition
 -> retrieve objective/VALD/curated candidates
 -> AI ranking or deterministic ranking
 -> validate and compose recommendation
 -> save customer_reco_form_data + assessment_context
 -> return recommended tests to dashboard
```

Main entry points:

- `context-layer/app/main.py`: `/health`, `GET /recommendations`, and
  `POST /recommendations/refresh`.
- `context-layer/app/service.py`: complete orchestration and cache decision.
- `context-layer/app/pipeline/`: classification, retrieval, ranking, composing,
  context and output-writing stages.
- `context-layer/data/tests_pool.json`: condition-to-test pool.

It reads `customer-info`, `objectiveAssessments`, and VALD data, and owns
`customer_reco_form_data` and `assessment_context`. Recommendation generation is
lazy: the dashboard calls it; it is not automatically invoked by the interview.

## 10. Main files to open during a client screen-share

| Client asks… | Open this file first |
|---|---|
| “Where does the frontend start?” | `../customer-agent-frontend/src/App.tsx`, then `pages/Index.tsx` |
| “Where is the main patient UI?” | `../customer-agent-frontend/src/components/TranscriptionInterface.tsx` |
| “How does WebSocket communication work?” | `../customer-agent-frontend/src/hooks/useWebSocket.ts`, then `DataHandling/server.py` |
| “Where do API/WS URLs come from?” | `../customer-agent-frontend/src/config/api.ts` and `config/policy.ts` |
| “Where are the APIs?” | `DataHandling/server.py` near the `@app.get/post/websocket` routes |
| “Where are intake questions configured?” | `DataHandling/data/forms/FRM-01.md` |
| “How is the Markdown form loaded?” | `DataHandling/src/forms/loader.py` |
| “How is the next question selected?” | `DataHandling/src/graph/nodes/generate.py` |
| “How is an answer saved into form fields?” | `DataHandling/src/graph/nodes/extract.py` and `pure_functions/form_extraction.py` |
| “What is the complete state machine?” | `DataHandling/src/graph/graph.py`, `edges.py`, and `state.py` |
| “Where are prompts?” | `DataHandling/src/prompts.py` |
| “Which AI models are used?” | `DataHandling/app/ai/models.py` |
| “How does speech-to-text work?” | `DataHandling/app/audio/stt.py` |
| “Where is MongoDB saving/loading?” | `DataHandling/server.py::save_customer_info/fetch_form_by_id` and `app/db/` |
| “How are reports processed?” | `DataHandling/server.py` attachment route, then `app/jobs/reports.py` and `docscanner/` |
| “What is the context-layer?” | `context-layer/app/service.py`, then `context-layer/app/pipeline/` |

## 11. Five-minute demo order

1. Show a patient URL and explain `userId`, `formId`, and optional `attemptId`.
2. Show the consent check and WebSocket connection in the browser Network tab.
3. Open `FRM-01.md` and explain that it defines what information must be
   collected.
4. Open `graph.py` and `generate.py` to show how the graph extracts, validates,
   and chooses the next question.
5. Give one typed answer and one voice answer; show `text_input`/audio WebSocket
   frames and the returned `text_message`.
6. Open MongoDB `customer-info` and show `formId`, `attemptId`, `form_data`,
   `status`, and timestamps using non-production/test data.
7. If reports are in scope, show the upload endpoint and durable job flow.
8. If recommendations are in scope, explain that context-layer is a separate
   dashboard-triggered downstream service.

## 12. One-minute client explanation

> The Customer Agent collects a patient's MSK intake through a React interface
> using text or voice. The frontend checks the existing consent record and opens
> a WebSocket to FastAPI. Voice is transcribed, then LangGraph extracts structured
> form data, checks what is missing, and asks the next relevant question using the
> form definition and conversation history. Every attempt is stored separately in
> MongoDB. Reports follow a validated S3 and durable Bedrock-processing flow. A
> separate context-layer can later read the completed intake and recommend
> objective assessment tests when requested by the clinician dashboard.
