# Stance Health Customer Agent — Complete Project Technical Reference

**Document status:** Current repository assessment  
**Assessment date:** 11 August 2026  
**Backend branch inspected:** `prod`  
**Frontend branch inspected:** `dev`  

## 1. Project purpose

The Stance Health Customer Agent is a patient-facing AI medical-intake application. It conducts a structured musculoskeletal intake through text and voice, converts free-form patient responses into a fixed clinical form, stores progress, supports report uploads, and presents the result for confirmation.

The product is an intake and information-collection system. It is not a diagnosis engine, treatment-prescription system, clinician dashboard, appointment system, or general electronic health record.

### Core capabilities

- Patient-specific assessment links.
- Consent status checking.
- Text conversation over WebSocket.
- Browser microphone capture and server-side speech-to-text.
- AI extraction of structured medical fields from natural language.
- Follow-up generation for missing information.
- Interview progress and section status.
- Resume of previously saved forms.
- Correction handling and summary confirmation.
- PDF/image medical-report upload.
- S3 storage and Bedrock-based document analysis.
- Optional text-to-speech responses.
- Basic clinic-information and medical-education responses.
- Health, Prometheus, structured-log, and optional Langfuse observability.

## 2. Repository layout

The solution is split into two sibling repositories:

```text
customerAgent/
├── customer-agent/             Backend repository
│   ├── DataHandling/
│   │   ├── server.py           Active FastAPI entry point and monolithic runtime
│   │   ├── app/                Partially extracted modular backend
│   │   ├── src/graph/          Active LangGraph interview workflow
│   │   ├── src/llm/            Legacy HealthAgent and LLM utilities
│   │   ├── src/prompts.py      Form schema and prompt definitions
│   │   ├── upload/             S3 integration
│   │   ├── docscanner/         Bedrock document analysis
│   │   ├── deployment/         Container and EC2 deployment material
│   │   ├── docs/               Older focused documentation
│   │   └── tests/              Smoke and LLM comparison scripts
│   ├── docker-compose.yml      Compose file used from the backend root
│   └── PROJECT_KT_OVERVIEW.md  This document
│
└── customer-agent-frontend/    Frontend repository
    ├── src/
    │   ├── App.tsx             Router and application providers
    │   ├── pages/               Route-level screens
    │   ├── components/          Interview UI and shared components
    │   ├── hooks/               WebSocket, voice, patient and appointment hooks
    │   ├── config/api.ts        Customer-agent HTTP/WebSocket configuration
    │   └── utils/               Separate GraphQL API client
    ├── public/                  Brand and static assets
    ├── vite.config.ts           Vite development configuration
    └── vercel.json              Production HTTP rewrites and SPA fallback
```

The repositories have independent Git histories, dependencies, environment files, builds, and deployments.

## 3. System architecture

```text
Patient browser
  │
  ├── HTTPS/HTTP requests ─────────────────────────────────────────┐
  ├── WebSocket control and text messages ──────────────────────┐  │
  └── WebSocket binary audio chunks ──────────────────────────┐ │  │
                                                               │ │  │
React 18 + TypeScript + Vite frontend                           │ │  │
  │                                                            │ │  │
  ├── Customer-agent API client                                │ │  │
  ├── WebSocket hook                                            │ │  │
  ├── MediaRecorder/Web Speech browser APIs                     │ │  │
  ├── Dynamic question controls                                 │ │  │
  └── Separate Stance GraphQL client                            │ │  │
                                                               ▼ ▼  ▼
FastAPI + Uvicorn backend
  │
  ├── WebSocket session controller
  │     ├── per-connection client_state
  │     ├── per-connection HealthAgent instance
  │     ├── LangGraph state adapter
  │     ├── text/audio protocol handling
  │     └── background TTS and persistence work
  │
  ├── LangGraph interview workflow
  │     ├── Gemini extraction
  │     ├── intent/correction/report classification
  │     ├── form validation
  │     ├── question generation
  │     └── summary/confirmation
  │
  ├── Faster-Whisper base model ── speech-to-text
  ├── Edge TTS / legacy gTTS ───── text-to-speech
  ├── MongoDB Atlas ─────────────── users, consent and forms
  ├── AWS S3 ───────────────────── private medical documents
  ├── Amazon Bedrock Nova ───────── report interpretation
  ├── ChromaDB ──────────────────── optional local knowledge retrieval
  ├── Langfuse/OpenInference ────── optional LLM traces
  └── Prometheus/structured logs ── runtime observability
```

### Runtime topology

| Component | Local address | Production role |
|---|---|---|
| Vite frontend | `http://localhost:8080` | Separately hosted frontend, currently designed around Vercel |
| FastAPI backend | `http://localhost:8000` | Containerized service behind `customeragent.stance.health` |
| WebSocket | `ws://localhost:8000/ws/{clientId}` | Direct secure WebSocket to backend domain |
| MongoDB | Remote URI from environment | Users, consent references, interview forms |
| S3 | AWS environment configuration | Medical-report object storage |
| Bedrock | AWS region/model configuration | Document summarization |
| ChromaDB | `/app/db/vector/...` | Optional brand/medical retrieval |

HTTP and WebSocket deployment paths differ. Vercel can rewrite normal HTTP routes, but the frontend intentionally connects WebSockets directly to the backend because Vercel rewrites are not a dependable WebSocket proxy.

## 4. Technology stack

### Frontend

- React 18.3.
- TypeScript 5.8.
- Vite 5.4 with SWC.
- React Router 6.
- Tailwind CSS 3.
- shadcn-style Radix UI components.
- TanStack React Query.
- Apollo Client and raw GraphQL fetch helpers.
- Browser MediaRecorder, AudioContext and optional Web Speech APIs.
- Sonner and Radix toast systems.

### Backend

- Python 3.12.
- FastAPI 0.115.
- Uvicorn.
- Native FastAPI WebSockets.
- LangGraph.
- LlamaIndex Core with Google GenAI adapter.
- Gemini 2.5 Flash Lite and Gemini 2.5 Flash.
- Faster-Whisper with CTranslate2.
- PyMongo.
- Boto3 for S3 and Bedrock.
- ChromaDB.
- Pydub, FFmpeg, Edge TTS and gTTS.
- Prometheus, structlog, OpenTelemetry and optional Langfuse.

## 5. Frontend architecture

### 5.1 Application providers

`src/App.tsx` creates:

- `QueryClientProvider` for TanStack Query.
- Tooltip provider.
- Two notification systems: the shadcn toaster and Sonner.
- `BrowserRouter` and application routes.

### 5.2 Routes

| Route | Behavior |
|---|---|
| `/` | Shows “Access via your link” because no patient ID exists |
| `/{userId}` | Redirects to `/{userId}/FRM-01` |
| `/{userId}/{formId}` | Opens the main interview UI |
| `/consent/{userId}` | Redirects to external `consent.stance.health` |
| `/{formKey}/{patientId}/{appointmentId}` | Opens older appointment-oriented `FormPage` flow |
| `*` | Not-found page |

`Index.tsx` accepts either clean path parameters or the older query format:

```text
?userId={id}&formId={id}
```

Query parameters are converted into a clean path. If no form ID is supplied, `FRM-01` is used.

### 5.3 Main interview component

`TranscriptionInterface.tsx` is the frontend production center. It is approximately 2,100 lines and manages:

- Conversation messages.
- Streaming LLM tokens.
- WebSocket status.
- Current transcript and edited transcript.
- Voice recording and processing state.
- Server-sent audio playback.
- Consent state.
- Current form and progress state.
- Attachment prompts and uploads.
- Agent thought-stream display.
- Welcome/input-mode selection.
- Dynamic question controls.
- Session ending and UI reset.

This component currently has too many responsibilities and is a primary frontend maintenance risk.

### 5.4 WebSocket hook

`useWebSocket.ts` owns:

- Connection creation and status.
- Reconnection after three seconds.
- Parsing JSON responses.
- Handling binary/Blob responses.
- Dispatching typed callbacks.
- Sending text, audio and session-control messages.
- Client-side connection cleanup.

The WebSocket URL used by the interview is:

```text
{VITE_WS_URL}/ws/{userId}
```

The URL client ID and the `userId` in `start_interview` happen to be the same in the current frontend, but the backend treats them as separate values.

### 5.5 Voice capture

`useVoiceRecorder.ts`:

- Requests microphone access.
- Chooses a supported browser recording MIME type.
- Captures chunks through `MediaRecorder`.
- Uses an analyser for audio-level animation.
- Optionally uses browser speech recognition for live interim text.
- Stops media tracks and audio contexts during cleanup.

The server transcription is authoritative. Browser live transcription is primarily a user-experience aid.

### 5.6 Consent behavior

The main interface calls:

```text
GET /api/users/{userId}/consent
```

In development mode, consent is bypassed. In production, a patient who has not consented is sent to the external consent application in another tab. When the interview tab becomes visible again, consent is checked again.

`ConsentPage.tsx` contains a complete local consent UI, but it is not routed by `App.tsx`; `/consent/{userId}` redirects externally instead. It is therefore currently unused route code.

### 5.7 Dynamic question controls

The UI contains renderers for:

- Short text and paragraph.
- Single choice and multiple choice.
- Checkbox.
- Yes/no.
- Number stepper.
- Dropdown.
- Date and time.
- Rating stars.
- Slider and linear scale.
- Likert scale.
- Choice grid and checkbox grid.
- File upload.
- Multi-answer grouped questions.
- PROM-oriented steps.

These rely on `question_meta`, `promSteps`, question IDs, per-question types, options, and scales. The active backend does not consistently emit this contract, so much of the dynamic renderer is presently dormant or only useful with another backend version.

### 5.8 Separate GraphQL integration

The frontend includes an independent GraphQL client for the wider Stance platform. It can retrieve:

- Centers.
- Patients.
- Appointments.

This GraphQL API is not provided by the customer-agent backend. It uses a separate domain, API key and organization header.

The patient/appointment hooks and `FormPage` represent an older clinician-selection workflow. The primary patient-link route does not require these hooks.

## 6. Backend startup and runtime

The container runs:

```text
uvicorn server:app --host 0.0.0.0 --port 8000
```

Importing `server.py` performs substantial startup work:

1. Loads environment variables.
2. Configures structured logging and Prometheus instruments.
3. Creates the FastAPI app and CORS middleware.
4. Initializes optional Langfuse instrumentation.
5. Loads the Faster-Whisper `base` model.
6. Instantiates the legacy `HealthAgent`.
7. Creates the Gemini reasoning model.
8. Builds the LangGraph workflow.
9. Connects to MongoDB and creates indexes.

Because model loading occurs during module import, startup can be slow and the first build/start may download Whisper model files from Hugging Face.

### Active runtime source of truth

The active Uvicorn entry point is `DataHandling/server.py`. The production behavior is not fully delegated to `DataHandling/app/`.

| Area | Active implementation | Modular alternative | Status |
|---|---|---|---|
| FastAPI app/routes | `server.py` | `app/routes/` | Modular health route exists but is not mounted |
| WebSocket handler | `server.py` | `app/ws/handler.py` | Modular handler is not registered |
| Start interview | `server.py` branch | `app/ws/stages/start_interview.py` | Duplicate; modular fixes do not affect runtime |
| Text processing | `server.py` branch | `app/ws/stages/text_input.py` | Duplicate with behavioral differences |
| Form repository | `server.py` functions | `app/forms/repository.py` | Duplicate |
| Progress calculation | `server.py` functions | `app/forms/progress.py` | Duplicate |
| Mongo connection | `server.py` globals | `app/db/mongo.py` | Duplicate clients/config |
| TTS | Legacy gTTS function plus modular Edge TTS | `app/audio/tts.py` | Graph path schedules modular Edge TTS; legacy path remains |
| STT | `server.py` global model | `app/audio/stt.py` | Duplicate loading abstraction |
| Sanitization/rate limit | `server.py` | `app/interview/` | Duplicate |

The modular code contains several improvements not present in the active monolithic handler. Code changes must verify which path is registered before assuming a fix is live.

## 7. Medical intake data model

The default form ID is fixed:

```text
FRM-01
```

Each patient can have one current form under the compound identity `(userId, formId)`.

### Form sections and fields

| Section | Fields |
|---|---|
| Present Complaint | Primary Complaint; Duration of the Issue; Onset; Mechanism/Cause |
| Previous Consultations | Previous Diagnosis/Advice/Treatment; Current Status |
| Pain Assessment | Primary Location; Severity 1–10; Aggravating Factors; Relieving Factors |
| History & Diagnostics | Illness/Surgical History; Current Lifestyle; Reports |
| Treatment Goals | Short-Term Goals; Long-Term Goals; Specific Expectations |
| Referral | Source |

There are 17 fixed fields across six sections.

The template is defined in `src/prompts.py`. `get_medical_form_template()` returns a deep copy to reduce cross-user state leakage. The public `MEDICAL_FORM_TEMPLATE` constant remains for backward compatibility and should not be mutated.

### Conditional completeness rules

- If the patient has never had a previous consultation, the previous-status field is not required.
- Negative history/report answers can be valid completed values.
- Generic referral values such as “yes,” “no,” or “none” do not complete the referral source.
- General-assessment visits can mark complaint/pain fields not applicable.
- Summary phase reports at least 85% progress; complete phase reports 100%.

## 8. LangGraph workflow

### 8.1 State

`InterviewState` carries:

- Identity: user, form and session IDs.
- Current input.
- Full form object.
- Question round and current section.
- Phase.
- Visit context.
- History.
- Report/upload flags.
- Correction and summary intent.
- Missing fields.
- Optional orchestrator question override.
- Response text and attachment request.

### 8.2 Phases

```text
welcome
interviewing
awaiting_upload
summary
complete
```

### 8.3 Graph nodes

| Node | Responsibility |
|---|---|
| `handle_first_turn` | Classifies the first message, initializes visit context and starts intake |
| `extract_form_data` | Extracts form fields, report intent and correction signals |
| `classify_intent` | Compatibility no-op; classification was merged into extraction |
| `validate_section` | Finds required missing fields |
| `advance_section` | Moves to the next incomplete section |
| `detect_correction` | Determines what existing information is being changed |
| `apply_correction` | Updates relevant form fields |
| `generate_question` | Produces the next targeted question |
| `handle_upload_response` | Manages report availability/upload intent |
| `generate_summary` | Creates final review text |
| `classify_summary_intent` | Classifies confirmation, rejection or correction |
| `handle_summary_response` | Completes, corrects, or reopens the interview |

### 8.4 Routing

```text
phase=welcome
  └─ handle_first_turn ── END for current turn

phase=interviewing
  └─ extract_form_data
       └─ classify_intent
            ├─ correction → detect_correction → apply_correction
            ├─ report flow → handle_upload_response
            └─ normal → validate_section
                         ├─ incomplete → generate_question
                         └─ complete → advance_section
                                       ├─ more sections → generate_question
                                       └─ all complete → generate_summary

phase=summary
  └─ classify_summary_intent → handle_summary_response
       ├─ confirm → complete
       ├─ correction → apply_correction
       ├─ new information → interviewing
       └─ unclear → summary again
```

### 8.5 Persistence boundary

The graph does not currently use a checkpointer. Full state is reconstructed from `client_state` on each turn. MongoDB is treated as the persistent form source, while mid-node graph state is not durable.

After a response is sent, form saving is scheduled in a background thread. This improves perceived latency but introduces durability and ordering risks.

## 9. AI model usage

### Gemini 2.5 Flash Lite

Used for most low-latency operations:

- Conversational phrasing.
- Intent classification.
- Follow-up generation.
- Summary handling.
- Corrections and fallbacks.
- Brand/education answers.

`HealthAgent.llm_complete()` rotates across fallback Gemini models on transient quota/service errors.

### Gemini 2.5 Flash

A separate reasoning-grade model is used for form extraction. Its purpose is to improve multi-field interpretation of long patient responses while limiting the more expensive model to extraction work.

### Prompt architecture

Prompts include:

- Welcome and first-turn conductor.
- Intelligent follow-up generation.
- Fixed medical schema.
- Reasoning extraction.
- Legacy formatting fallback.
- Correction detection/application.
- Summary generation/classification.
- Report-intent classification.

Three large predefined question groups remain as fallbacks. They ask many questions at once and are the source of the long multi-question messages seen during testing.

### Visit context

The extractor classifies:

- `specific_complaint`
- `general_assessment`
- `clinic_inquiry`
- `unknown`

This classification controls whether complaint and pain questions are required.

### Off-topic routing

The WebSocket layer intercepts apparent:

- Stance Health questions.
- Medical education questions.
- Requests for a preliminary assessment.

These are answered outside the graph so they do not alter the form. The active monolithic classifier is broad and can incorrectly classify normal intake answers containing phrases such as “Stance Health.” The modular classifier is more restrictive but is not wired into the active endpoint.

## 10. Complete patient flow

```text
1. Patient opens /{userId}/FRM-01.
2. Frontend checks consent.
3. Frontend opens /ws/{userId}.
4. Frontend sends start_interview with userId and formId.
5. Backend validates the user in MongoDB.
6. Backend loads FRM-01 or initializes a fresh in-memory form.
7. Backend restores history when available.
8. Backend sends welcome/next question.
9. Patient submits text or recorded audio.
10. Audio is transcribed and returned as text.
11. Submitted text is sanitized and rate-limited.
12. Off-topic routing runs before the graph.
13. LangGraph extracts structured fields.
14. Required fields are validated.
15. The next question or summary is generated.
16. Response tokens and final message are sent.
17. TTS may be generated in the background.
18. Form data is saved in a background thread.
19. Steps 9–18 repeat.
20. Patient confirms or corrects the summary.
21. Phase becomes complete.
```

## 11. WebSocket protocol

### Endpoint

```text
/ws/{client_id}
```

### Inbound messages

| Type | Important fields | Behavior |
|---|---|---|
| `start_interview` | `userId`, optional `formId`, timestamp | Validate user and start/resume |
| `start_new_form` | `userId` | Reset session; constrained by fixed form model |
| `load_form` | `formId` | Load a form belonging to current user |
| `text_input` | `text`, timestamp | Process one conversational turn |
| `audio_start` | timestamp | Reset audio buffers and begin capture |
| Binary frame | raw audio bytes | Append to current recording |
| `audio_end` | duration, timestamp | Decode and transcribe accumulated audio |
| `end_session` | `userId`, timestamp | Save state and end/pause session |

The frontend has an `audio_chunk` JSON concept in its type history, but the active browser flow sends audio as binary WebSocket frames.

### Outbound messages

| Type | Important fields | Behavior |
|---|---|---|
| `token` | `content` | Incremental response text |
| `text_message` | text plus interview state | Final assistant response |
| `thought_update` | stages/statuses | UI processing visualization |
| `transcription` | transcribed text | Result of server STT |
| `form_loaded` | form data/state | Resume payload |
| `chat_history` | messages | Restored conversation |
| `form_selection_required` | forms | Older multi-form flow |
| `audio_start` | ID and byte size | Start server TTS stream |
| `audio_chunk` | base64 chunk and completion flag | Server TTS data |
| `error` | `message` or `text` | Validation/runtime error |
| `system` | message | Graceful-shutdown notification |

The error contract is inconsistent: some branches use `message`, others use `text`. The frontend accounts for some but not all variations.

## 12. Audio processing

### Speech-to-text

- Model: Faster-Whisper `base`.
- CPU compute: `int8`.
- GPU compute: `float16` when `nvidia-smi` is found.
- Expected sample rate: 16 kHz.
- Mono, 16-bit output for internal WAV processing.
- FFmpeg/Pydub handle WebM/Opus and other browser formats.

The full model is loaded at startup. This increases cold-start time and memory but avoids per-request initialization.

### Text-to-speech

Two implementations coexist:

1. A legacy synchronous gTTS function in `server.py`, including WAV caching and speed-up.
2. The modular Edge TTS implementation using Indian English voice with an American English fallback.

The newer graph path schedules Edge TTS in a background task. The unconditional legacy `gtts` import caused the earlier container crash when `gTTS` was absent from Docker requirements.

### Audio reliability concerns

- Audio buffers live in process memory.
- No explicit total WebSocket recording-size ceiling is enforced before decoding.
- TTS runs after the text reply and can attempt to write to a closed socket.
- Browser and server transcription paths create complex duplicate-submission behavior; an eight-second deduplication heuristic exists.

## 13. HTTP API

| Method | Endpoint | Purpose | Authentication in this service |
|---|---|---|---|
| GET | `/health` | Component health | None |
| GET | `/metrics` | Prometheus metrics when enabled | None |
| GET | `/docs` | OpenAPI UI | None |
| GET | `/api/users` | Search/list users | None |
| GET | `/api/users/{userId}/consent` | Read consent state | None |
| POST | `/api/users/{userId}/consent` | Write consent state | None |
| GET | `/api/users/{userId}/forms` | List user forms | None |
| GET | `/api/forms/{formId}?userId=...` | Read form | None |
| GET | `/api/forms/{formId}/progress?userId=...` | Read progress | None |
| POST | `/api/forms/{formId}/attachments` | Upload reports | None beyond form/user matching |

The backend has no JWT/session/API-key middleware. URL possession and user IDs are currently treated as identity context, not strong authentication.

## 14. MongoDB architecture

### Collections

| Collection | Purpose |
|---|---|
| `users` | Existing Stance user directory and patient validation |
| `customer-info` | Intake forms, progress, history and attachments |
| `consentrecords` | External consent records read as a secondary source |

### Form document

```json
{
  "_id": "Mongo ObjectId",
  "userId": "ObjectId or legacy string",
  "formId": "FRM-01",
  "title": "Generated from primary complaint",
  "timestamp": "ISO string",
  "form_data": {
    "Present Complaint": {},
    "Previous Consultations": {},
    "Pain Assessment": {},
    "History & Diagnostics": {},
    "Treatment Goals": {},
    "Referral": {}
  },
  "current_section": "section name",
  "attachments": [],
  "createdAt": "Mongo date",
  "updatedAt": "Mongo date"
}
```

### Indexes

- Unique compound index on `(userId, formId)`.
- Partial TTL index on `createdAt` for documents titled `New Form`, expiring after seven days.

### Save strategy

- IDs are converted to `ObjectId` where possible while legacy strings remain supported.
- Form data is deep-copied before persistence.
- `update_one(..., upsert=True)` is used with `$setOnInsert` for `createdAt`.
- Existing attachments are preserved unless explicitly supplied.
- A title may require an additional LLM call when a primary complaint exists.
- Chat history support exists in signatures/state, but persistence and restoration behavior is not consistently implemented across active and modular paths.

### Current environment safety

The local backend environment inspected during this assessment connects to a production-named MongoDB Atlas cluster and the `stance-dashboard` database. Local UI testing with a valid user can therefore create or update real `customer-info` records. A separate development database is required for safe end-to-end testing.

## 15. Medical-report processing

### Upload sequence

```text
Frontend selects one or more files
  → multipart POST with userId
  → backend loads form and verifies ownership
  → backend reads each full file into memory
  → 15 MB per-file size check
  → upload to S3
  → append attachment records in MongoDB
  → mark Reports as processing
  → return HTTP response immediately
  → background Bedrock analysis
  → update Reports field with findings/measurements/impression
```

### Supported analysis formats

- PDF, converted to page images using Poppler/pdf2image.
- PNG.
- JPG/JPEG.
- WebP.

### Attachment record

```json
{
  "id": "UUID",
  "type": "report",
  "label": "Medical Report",
  "fileName": "original name",
  "url": "S3 URL",
  "uploadedAt": "ISO timestamp",
  "contentType": "reported MIME type"
}
```

### Bedrock result

The document scanner attempts to return structured findings, measurements and an impression. The result is converted into human-readable text and stored under `History & Diagnostics.Reports`.

Background tasks are process-local. A container restart after upload can lose unfinished analysis work, and there is no durable queue/retry record.

## 16. Configuration reference

### Backend variables

| Variable | Required | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | Yes for AI | Primary Gemini credential |
| `GOOGLE_API_KEY` | Alternative | Gemini fallback name |
| `GOOGLE_GEMINI_API_KEY` | Alternative in server | Additional fallback name |
| `MONGO_URI` | Yes for real sessions | MongoDB connection URI |
| `MONGO_DB_NAME` | Optional | Defaults to `stance-dashboard` |
| `MONGO_USERS_COLLECTION` | Optional | Defaults to `users` |
| `AWS_ACCESS_KEY_ID` | Required for S3 | AWS credential |
| `AWS_SECRET_ACCESS_KEY` | Required for S3 | AWS credential |
| `AWS_S3_REGION` | Required/recommended | S3 region |
| `S3_BUCKET_NAME` | Required for upload | S3 bucket |
| `AWS_REGION` | Required for Bedrock | Bedrock region |
| `BEDROCK_MODEL_ID` | Optional | Defaults to Amazon Nova Lite identifier |
| `BEDROCK_INFERENCE_PROFILE` | Region-dependent | Bedrock inference profile |
| `CHROMA_DB_PATH` | Optional | Local vector-store path |
| `LANGFUSE_PUBLIC_KEY` | Optional | LLM tracing |
| `LANGFUSE_SECRET_KEY` | Optional | LLM tracing |
| `LANGFUSE_BASE_URL` | Optional | Langfuse host |
| `HF_TOKEN` | Optional | Faster Hugging Face model downloads |

Names such as `AWS` or `S3` do not satisfy the code. The exact variables above are required.

The Compose file also sets Google credential file paths, but the active Gemini path primarily uses API-key authentication. Missing JSON credential files are not fatal for that path.

### Frontend variables

| Variable | Purpose |
|---|---|
| `VITE_API_URL` | Customer-agent HTTP base URL |
| `VITE_WS_URL` | Customer-agent WebSocket base URL |
| `VITE_API_KEY` | Separate platform GraphQL credential |
| `VITE_GRAPHQL_URL` | Separate platform GraphQL endpoint |

### Vite environment precedence

For `npm run dev`, Vite gives development-specific files priority:

```text
.env.development.local
.env.local
.env.development
.env
```

The repository's `.env.development` points to a remote development host. A local `.env.development.local` is therefore needed to force `localhost:8000`. Local files are ignored by Git.

## 17. Deployment

### Backend container

The Docker image:

- Uses `python:3.12-slim`.
- Installs FFmpeg, libsndfile, Git, curl, build tools and Poppler.
- Installs `requirements-docker.txt`.
- Copies `DataHandling` to `/app`.
- Creates runtime directories.
- Exposes port 8000.
- Runs a `/health` Docker health check.
- Starts Uvicorn.

### Compose behavior

The root Compose file:

- Builds the backend image.
- Publishes port 8000.
- Loads `DataHandling/.env`.
- Mounts the complete `DataHandling` directory over `/app` read-write.
- Mounts runtime/audio/vector/config directories.
- Restarts unless stopped.
- declares CPU and memory limits/reservations.

Mounting all source over `/app` means the image's copied code is hidden at runtime. This enables rapid updates but weakens image immutability and makes host files production-critical.

There is a second Compose file under `DataHandling/deployment/`. It still declares obsolete Compose `version: '3.8'` and has context/path assumptions that deployment scripts rewrite.

### EC2 scripts

The deployment directory contains scripts for code packaging, rsync/SCP, SSH, rebuild/restart, bytecode cleanup and health checks. They contain a fixed EC2 hostname, fixed username, expected PEM filename and fixed remote directory.

These scripts are environment-specific operational automation, not portable deployment tooling.

### Frontend deployment

`vercel.json`:

- Rewrites `/health` and `/api/*` to the backend domain.
- Includes a `/ws/*` rewrite even though application code directly connects to the WebSocket backend.
- Rewrites all remaining routes to `index.html` for SPA deep links.

## 18. Observability and operations

### Health

`/health` returns:

- MongoDB: `ok` or `degraded`.
- Graph: `ok` or `degraded`.
- Whisper: always reported as `ok` after successful import/startup.

### Prometheus

Declared metrics include:

- LLM calls.
- LLM latency.
- WebSocket active connections.
- Model fallbacks.
- Interview turns.

The metrics ASGI app is mounted at `/metrics` when dependencies import successfully.

### Logging

- Structlog produces JSON for selected lifecycle events.
- Much of the application still uses `print`.
- Logs include user IDs, patient text, extracted medical fields, model outputs and form state.
- Correlation-ID tooling is installed but not consistently applied across all log output.

### Langfuse

Optional instrumentation traces LlamaIndex calls when both credentials are set. Healthcare data sent to observability platforms requires an explicit privacy, retention and contractual review.

### Graceful shutdown

The lifespan handler:

- Marks the service shutting down.
- Sends a system message to active sockets.
- Waits up to 15 seconds for sessions to close.

Background saves, OCR and TTS tasks are not centrally drained or durably queued.

## 19. Testing status

### Existing backend tests

`tests/smoke.py` checks:

- `/health` returns 200.
- `/api/users` is wired.
- WebSocket accepts a connection.
- Missing `userId` produces a structured error.

`tests/llm_ab.py` compares model response, tokens, latency and estimated cost for two sample prompts.

### Missing coverage

- No frontend unit tests.
- No frontend component tests.
- No committed end-to-end browser tests.
- No complete LangGraph node/edge unit suite.
- No extraction regression dataset.
- No correction-flow regression suite.
- No report upload/OCR integration test.
- No authorization tests.
- No WebSocket reconnection/resume tests.
- No concurrency/cross-user isolation stress test.
- No background-save ordering test.
- No production-like load test.
- No medical safety/red-flag test suite.

The smoke test does not cover an existing-user session and therefore does not detect most interview-loop failures.

## 20. Question-management status

There is no database-backed question-management system in this repository.

Missing management capabilities:

- Question collection/table.
- Category collection/table.
- Create, edit, delete and reorder APIs.
- Administrative UI.
- Enable/disable and version controls.
- Conditional branching editor.
- Tenant/organization-specific question sets.
- Audit history and publishing workflow.
- Localization.

Current behavior is:

```text
Fixed 17-field form
  → detect missing fields
  → generate conversational question with Gemini
  → extract answer back into fixed fields
```

Three predefined multi-part question groups are embedded in prompts as fallback content. These are not independently manageable data records.

The frontend contains a much broader structured-question renderer, but the current backend does not provide a complete, versioned metadata contract to drive it.

## 21. Confirmed defects and risks

### Critical

#### C1. Local environment can write to production data

The inspected environment points at a production-named Atlas cluster. Starting an interview with a valid patient automatically creates or updates `customer-info`. Development and production are not safely isolated by default.

**Required action:** Create separate development/staging clusters or databases, use unmistakable environment names, deny production credentials in local setups, and seed synthetic users.

#### C2. No authentication/authorization enforcement

User IDs in URLs and request parameters control access. `/api/users`, form reads, consent writes and WebSocket sessions have no backend authentication middleware.

**Impact:** ID disclosure or guessing may expose or alter health data.

**Required action:** Validate signed short-lived assessment tokens or authenticated sessions on every HTTP and WebSocket operation; enforce organization and patient ownership server-side.

#### C3. Frontend contains a browser-visible platform API key

The GraphQL API key has both an environment path and a hardcoded fallback. The organization ID is also hardcoded. Any Vite value is delivered to the browser and must not be treated as secret.

**Required action:** Rotate the exposed key, remove fallback credentials, use an authenticated backend/BFF or strictly scoped public credential, and derive organization from trusted identity.

#### C4. Patient health information is logged

Current logs print patient answers, extracted form contents, user IDs, LLM responses and processing details.

**Required action:** Introduce structured redaction, prohibit raw clinical content in default logs, restrict access, define retention, and audit third-party tracing.

### High

#### H1. Invalid-user WebSocket validation crashes

The active `start_interview` error branch evaluates a PyMongo `Collection` as a boolean. PyMongo rejects truth-value testing, so the intended “user does not exist” response fails. The socket remains able to send `text_input` messages without a valid session identity.

Observed behavior:

```text
Collection objects do not implement truth value testing
```

No database save occurred because `user_id` remained empty, but processing and LLM calls continued.

**Fix:** Compare collection objects with `is not None`, send the error, close or mark the session unauthorized, and reject every non-start message until session initialization succeeds.

#### H2. Active off-topic classifier misroutes valid intake answers

The monolithic handler classifies any long response containing `stance health` as a brand question. A referral answer such as “I found Stance Health online” skips form extraction and produces promotional text. Missing fields remain unchanged, creating a question loop.

The modular `text_input.py` has safer medical-content guards, but it is unused.

**Fix:** Run form-answer extraction before off-topic routing when an intake question is pending, classify questions rather than substrings, and port tests/fixes into the registered handler.

#### H3. Refactor duplication causes divergent behavior

The active WebSocket handler remains in `server.py`, while extracted handlers contain newer fixes. The same concerns exist for Mongo, forms, progress, audio, sanitization and messaging.

**Impact:** Fixes can be made in inactive files; behavior differs depending on path; defects return during merges.

**Fix:** Register modular routes/handler, add parity tests, then remove monolithic duplicates in controlled steps.

#### H4. Missing frontend module import

`App.tsx` imports `./pages/DirectFormPage`, but the file is absent and the symbol is unused. A clean TypeScript/Vite build can fail resolution.

**Fix:** Remove the import or restore and route the intended page; enforce clean builds in CI.

#### H5. Background persistence is not ordered or durable

Each turn can schedule an independent background save. Slow earlier saves may finish after later saves and overwrite newer state. Process termination can lose pending saves.

**Fix:** Serialize saves per `(userId, formId)`, include state version/turn number in conditional updates, and use awaited persistence at critical boundaries or a durable queue.

#### H6. Medical safety routing is incomplete

The code includes “no definitive diagnosis” prompt language but has no clear deterministic emergency/red-flag triage for urgent symptoms.

**Fix:** Define clinician-approved red flags, immediate escalation responses, emergency disclaimers, audit events, and tests outside general LLM prompting.

### Medium

#### M1. Frontend configuration is conflicting and environment-specific

`.env` targets localhost, `.env.development` targets a fixed remote IP, source code contains remote fallbacks, and production has separate proxy/direct rules. Restarting Vite can appear ineffective because mode-specific files override base settings.

**Fix:** Remove source-code IP fallbacks, provide `.env.example`, validate required values at startup, and document one configuration per environment.

#### M2. `FormSelection` calls progress without required `userId`

The backend requires `?userId=...`, but the older form-selection component calls `/api/forms/{formId}/progress` without it.

**Fix:** Include the parameter and add API-contract tests, or remove the unused older flow.

#### M3. Consent implementations are duplicated

An internal consent page exists, but routing redirects to the external consent product. Development bypasses consent entirely.

**Fix:** Select one ownership model, remove unused UI, and ensure test/staging can exercise real consent behavior safely.

#### M4. Upload validation trusts client metadata

The server reads entire files before size validation and does not strongly validate content signatures. Partial S3 uploads can remain if later files or database operations fail.

**Fix:** Stream with byte limits, inspect magic bytes, allowlist extensions/MIME, malware-scan, clean partial objects, and use upload status records.

#### M5. OCR background work has no durable lifecycle

There is no job ID endpoint, queue, retry/backoff record, dead-letter handling or restart recovery.

**Fix:** Use a durable worker queue and explicit attachment states: uploaded, queued, processing, completed, failed.

#### M6. Health endpoint can overstate readiness

Whisper is returned as `ok` unconditionally once the application imported, and Gemini/S3/Bedrock are not represented.

**Fix:** Separate liveness and readiness, expose dependency states without secrets, and check critical service initialization.

#### M7. In-memory state is instance-local

WebSocket state, locks, rate limits and graph state live in one process. Multiple workers/instances cannot transparently resume a live session.

**Fix:** Use sticky sessions plus durable state or externalize session/checkpoint and rate-limit state.

#### M8. Reconnection lacks strong session resumption semantics

The frontend reconnects after three seconds, but a new server connection receives a new in-memory session. Form data can reload from MongoDB, while exact pending phase/history may be lost.

**Fix:** Add signed session IDs, durable checkpoints, sequence numbers and idempotent start/resume behavior.

#### M9. Monolithic frontend component

The 2,100-line interview component combines protocol, media, consent, uploads, dynamic controls and rendering.

**Fix:** Split into a session controller hook, conversation store, audio controller, upload service, question renderer and presentation components.

#### M10. Monolithic legacy agent remains extremely large

`src/llm/functionalities.py` is approximately 3,900 lines and is still initialized as a fallback. It contains old and commented implementations alongside current methods.

**Fix:** Replace the fallback with small explicit services and delete superseded code after behavioral tests pass.

#### M11. Dependency definitions diverge

`requirements.txt` contains heavyweight GUI, Torch, old Whisper, Vertex, Redis and multiple LlamaIndex packages not used by the Docker runtime. `requirements-docker.txt` is leaner but still has broad version ranges.

**Fix:** Choose one dependency source, lock transitive versions, separate dev/test extras, and automate vulnerability/licence checks.

#### M12. TLS validation is weakened for MongoDB

The active connection uses `tlsAllowInvalidCertificates=True`.

**Fix:** Remove this flag, install the correct CA chain, and fail closed on certificate errors.

### Low/maintenance

#### L1. Large commented legacy block

The first approximately 430 lines of `server.py` are a commented older server implementation. This obscures navigation and can become stale misinformation.

#### L2. Duplicate and stale comments

Comments state MongoDB is disabled even though it is initialized and actively used. Other comments describe checkpointers or old behavior that no longer matches implementation.

#### L3. Duplicate Compose files

Root and deployment Compose files differ in health check, environment and obsolete-version usage.

#### L4. Unused or older frontend flows

Likely candidates include `ConsentPage`, older patient/appointment selection hooks, `FormSelection`, schema templates, onboarding components and numerous generic UI components. Usage should be confirmed by import analysis before deletion.

#### L5. Excessive browser console logging

The frontend logs user IDs, form IDs, connection state and API responses. Production logging should be minimized and redacted.

#### L6. Error payload inconsistency

WebSocket errors alternate between `text` and `message`. HTTP responses also use differing envelope shapes.

#### L7. Unbounded or unclear cache/retention policy

Legacy TTS cache and runtime folders need explicit size and retention controls. Medical documents, transcripts, chat history and observability traces also need formal retention rules.

#### L8. Documentation drift

Older documents refer to removed files, deterministic question pools, old model choices, missing deployment scripts and historical architecture. They should be marked historical or consolidated after this document is approved.

## 22. Legacy and unwanted-code register

| Item | Classification | Recommended disposition |
|---|---|---|
| Commented server implementation at top of `server.py` | Dead legacy | Delete after version-history confirmation |
| Monolithic WebSocket branches in `server.py` | Active legacy | Replace with `app/ws` after parity tests |
| `app/ws` modular handler | Intended replacement, currently inactive | Complete and register |
| Duplicate Mongo/form/progress/audio helpers | Transitional duplication | Consolidate behind one module per concern |
| `HealthAgent` large fallback surface | Active compatibility layer | Shrink, then remove |
| Legacy formatter prompts | Fallback | Retain only with measured regression value |
| Synchronous gTTS implementation/import | Mostly legacy | Remove when all paths use Edge TTS |
| Full `requirements.txt` | Legacy local environment | Replace with locked runtime/dev groups |
| Internal `ConsentPage` | Unused under current routing | Delete or make authoritative |
| `DirectFormPage` import | Broken/unused | Remove or implement |
| Appointment/patient selection UI | Older product flow | Confirm ownership, then isolate or remove |
| Frontend dynamic PROM renderer | Future/partial integration | Define backend contract or feature-flag |
| `form-schemas.ts` clinician schemas | Not part of active intake route | Confirm usage; remove if orphaned |
| Deployment scripts with fixed host/key names | Environment-specific legacy ops | Replace with CI/CD and parameters |
| Deployment Compose file | Duplicate configuration | Consolidate with root Compose |
| Older focused docs | Historical/stale mix | Archive with dates or merge authoritative facts |

## 23. Optimization roadmap

### Phase 0 — Safety and environment isolation

1. Create separate dev, staging and production MongoDB environments.
2. Seed synthetic test patients.
3. Rotate/remove frontend-exposed credentials.
4. Add authentication and authorization.
5. Remove Mongo invalid-certificate allowance.
6. Redact PHI from logs and tracing.

### Phase 1 — Correctness

1. Fix invalid-user WebSocket state and close unauthorized sessions.
2. Fix off-topic/form-answer routing.
3. Resolve missing frontend module.
4. Correct progress API parameters.
5. Standardize WebSocket and HTTP contracts.
6. Add deterministic emergency/red-flag handling.
7. Add regression tests for the observed interview loop.

### Phase 2 — Architecture consolidation

1. Wire `app/ws/handler.py` into FastAPI.
2. Move HTTP routes to `app/routes`.
3. Use one Mongo connection provider.
4. Use one form repository and progress calculator.
5. Use one audio implementation.
6. Split the frontend interview component.
7. Delete inactive duplicates only after parity verification.

### Phase 3 — Durability and scale

1. Add durable LangGraph checkpoints or an equivalent session store.
2. Add form state versions and ordered writes.
3. Move OCR to a durable queue/worker.
4. Externalize rate limits for multi-instance operation.
5. Add idempotency keys and message sequence IDs.
6. Define document/audio/chat retention and deletion workflows.

### Phase 4 — AI quality and cost

1. Build a de-identified extraction evaluation dataset.
2. Measure field accuracy, repetition rate, turns to completion and correction accuracy.
3. Replace substring off-topic routing with structured classification or graph-first logic.
4. Ask one or two related questions per turn instead of six-question batches.
5. Reduce repeated schema/history tokens in prompts.
6. Cache safe static prompt content where model economics justify it.
7. Avoid title-generation LLM calls when a deterministic title is adequate.
8. Track per-interview token use and model fallback frequency.

### Phase 5 — Delivery and operations

1. Add backend and frontend CI.
2. Require lint, type-check, clean build, unit tests and smoke tests.
3. Build immutable images without a full source-code bind mount in production.
4. Publish versioned images and perform staged rollout/rollback.
5. Replace fixed-host scripts with CI/CD secrets and environment parameters.
6. Add dashboards and alerts for latency, errors, disconnects, save failures and queue age.

## 24. Recommended target architecture

```text
Authenticated assessment link / signed token
  ↓
React route + session controller
  ├─ typed REST client
  ├─ versioned WebSocket client
  ├─ audio controller
  ├─ upload controller
  └─ question renderer
  ↓
FastAPI application
  ├─ auth/tenant middleware
  ├─ REST routers
  ├─ thin WebSocket protocol handler
  ├─ interview application service
  ├─ LangGraph domain workflow
  ├─ form repository with optimistic versioning
  ├─ durable session/checkpoint store
  ├─ durable OCR job queue
  └─ redacted observability
  ↓
Environment-isolated services
  ├─ MongoDB
  ├─ S3
  ├─ Bedrock
  ├─ Gemini
  └─ optional managed telemetry
```

The target design keeps clinical workflow logic separate from transport, model providers, persistence and presentation. It also makes every mutation authenticated, versioned and testable.

## 25. Operational procedures

### Start backend locally

```bash
cd ~/Desktop/stance/customerAgent/customer-agent
docker compose up -d --build
docker compose ps
docker compose logs -f healthflex-agent
```

Expected healthy response:

```json
{
  "status": "ok",
  "service": "healthflex-customer-agent",
  "components": {
    "mongodb": "ok",
    "graph": "ok",
    "whisper": "ok"
  }
}
```

### Start frontend locally

Create `customer-agent-frontend/.env.development.local`:

```dotenv
VITE_API_URL=http://localhost:8000
VITE_WS_URL=ws://localhost:8000
```

Then:

```bash
cd ~/Desktop/stance/customerAgent/customer-agent-frontend
npm install
npm run dev
```

Vite runs on port 8080.

### Safe test rule

Do not run a full interview against a valid production user from a local environment. A nonexistent ID can test socket establishment and rejection only. Full workflow tests require a synthetic user in a non-production database.

### Useful checks

```bash
docker compose config --quiet
docker compose ps
docker compose logs --tail=200 healthflex-agent
curl http://localhost:8000/health
curl http://localhost:8000/docs
```

### Dependency changes

Python dependency changes require an image rebuild. Source changes are currently visible through the bind mount but still require a process/container restart for module-level initialization changes.

### First startup

Allow additional time for the Faster-Whisper model download. `HF_TOKEN` is optional but can improve Hugging Face rate limits.

## 26. Data and privacy checklist

- Classify patient responses, transcripts, form data and reports as sensitive health information.
- Encrypt transport and storage.
- Use least-privilege MongoDB and AWS identities.
- Keep S3 objects private and use short-lived signed access where needed.
- Do not place reusable secrets in browser bundles.
- Redact application logs and model traces.
- Define consent evidence and policy versioning.
- Define retention and deletion for forms, audio, reports, caches, logs and traces.
- Record an audit trail for reads, changes, uploads, consent and completion.
- Validate third-party AI/observability data processing terms.
- Provide incident response and credential-rotation procedures.

## 27. Ownership boundaries

| Concern | Current owner/component |
|---|---|
| Patient interview UI | Customer-agent frontend |
| Patient/appointment directory | Separate Stance GraphQL platform |
| Consent experience | External consent application; backend also exposes consent APIs |
| User identity records | MongoDB `users` / surrounding platform |
| Intake conversation | FastAPI WebSocket + LangGraph |
| Structured intake record | MongoDB `customer-info` |
| Speech recognition | Local Faster-Whisper |
| Response speech | Edge TTS with legacy gTTS present |
| Report storage | AWS S3 |
| Report interpretation | Amazon Bedrock |
| General clinic knowledge | ChromaDB/Gemini fallback |
| Production hosting automation | Environment-specific EC2 scripts and Docker Compose |

## 28. Final technical assessment

The project has a functional core: the local container starts, MongoDB connects, Whisper loads, Gemini/LangGraph initializes, the health probe succeeds, and the browser can establish the WebSocket after correct environment configuration.

The main engineering constraint is not the AI model itself; it is architectural transition and safety. Production behavior still lives largely in a monolithic server while improved modular replacements remain inactive. This creates duplicated logic, inconsistent fixes and difficult verification. The absence of backend authentication, local access to production data, browser-visible credentials and raw health-data logging are the highest-priority risks.

The interview loop observed during testing is explained by two concrete active-path defects: invalid-user error handling fails on PyMongo truth testing, and brand-keyword routing can intercept a legitimate intake answer before extraction. No form was written during that dummy-user test because the session never acquired a valid user ID.

The immediate direction is: isolate environments, secure every boundary, fix the confirmed runtime defects, establish regression tests, connect the modular backend, then remove duplicated legacy code. Question management should be treated as a separate product capability requiring a defined schema, API, administrative workflow and versioned frontend/backend contract.
