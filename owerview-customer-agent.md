# Customer Agent — Runtime Flow Meeting Guide

Use this document during client calls. Each section explains what happens, where
it happens, and what to say in simple language.

## 1. Project in 30 seconds

> This is an AI-assisted medical intake application. The React frontend collects
> consent, text, voice answers, and reports. It communicates with a FastAPI backend
> using REST APIs and a WebSocket. The backend authenticates the patient, runs the
> interview through a LangGraph workflow, uses Gemini for conversation and clinical
> extraction, and saves each intake attempt in MongoDB. Voice is transcribed by
> Gemini first, with Google Cloud Speech-to-Text as fallback. Uploaded reports are
> stored in S3 and processed asynchronously.

## 2. Complete architecture

```text
Patient browser (React/Vite)
        |
        |-- REST: consent, forms, progress, uploads
        |-- WebSocket: interview, text, audio, live responses
        v
FastAPI backend
        |
        |-- Authentication and patient-ownership checks
        |-- LangGraph interview workflow
        |-- Gemini 2.5 Flash Lite / Gemini 2.5 Flash
        |-- Gemini audio -> Google Cloud STT fallback
        |-- S3 + report-processing worker
        v
MongoDB: users, customer-info, questions, attempts and jobs
```

Main files to open:

- Frontend routes: `customer-agent-frontend/src/App.tsx`
- Frontend screen: `customer-agent-frontend/src/components/TranscriptionInterface.tsx`
- Frontend WebSocket: `customer-agent-frontend/src/hooks/useWebSocket.ts`
- Backend entry/API: `DataHandling/server.py`
- Interview graph: `DataHandling/src/graph/graph.py`
- AI models: `DataHandling/app/ai/models.py`

## 3. Patient opens an interview link

```text
Patient link -> React route -> token read -> consent checked -> WebSocket opened
-> start_interview sent -> backend validates token and patient -> form returned
```

1. React matches `/{userId}/{formId}` in
   `customer-agent-frontend/src/App.tsx:38`.
2. The frontend reads `access_token` from the URL fragment in
   `customer-agent-frontend/src/config/auth.ts:3`.
3. It removes the token from the address bar and keeps it in session storage.
4. It checks consent through `GET /api/users/{userId}/consent` from
   `TranscriptionInterface.tsx:784`.
5. It opens the WebSocket in `useWebSocket.ts:151`.
6. It sends `start_interview` with the patient ID, form ID, attempt ID and token
   from `useWebSocket.ts:401`.
7. The backend receives it at `server.py:1988`, validates authentication and
   patient access, then loads or creates the interview state.
8. The backend returns the form, history and current state to the frontend.

Say to the client:

> The browser does not trust only the patient ID in the URL. It also supplies a
> signed access token, and the backend checks that the token is allowed to access
> that patient before returning information or starting the interview.

## 4. Consent flow

```text
Frontend checks consent -> backend reads patient consent -> accepted: continue
                                                -> missing: block interview start
```

- Consent GET endpoint: `DataHandling/server.py:4408`
- Consent update endpoint: `DataHandling/server.py:4469`
- Frontend consent gate: `TranscriptionInterface.tsx:784`

Say to the client:

> Consent is checked before enabling the interview. If it is not recorded, the
> patient is directed to the consent process. The UI fails closed if the consent
> API cannot be reached.

## 5. Where questions come from

There are three question sources—not one.

### Standard FRM-01

- Form sections/categories: `DataHandling/data/forms/FRM-01.md:7`
- Predefined questions: `DataHandling/data/forms/FRM-01.md:81`
- Loader: `DataHandling/src/forms/loader.py:149`
- Loaded exports: `DataHandling/src/prompts.py:103`

### AI follow-up questions

- First-turn instructions: `DataHandling/src/prompts.py:11`
- Intelligent follow-up instructions: `DataHandling/src/prompts.py:44`
- Question generation: `DataHandling/src/graph/nodes/generate.py:73`

### PROM/tagged questionnaires

- Retrieval: `DataHandling/server.py:791`
- MongoDB `tagged-questions`: `DataHandling/server.py:574`
- `question-bank` resolution: `DataHandling/server.py:878`
- Personalisation: `DataHandling/server.py:1059`

Say to the client:

> The standard FRM-01 structure is maintained in a Markdown definition. The AI
> uses the missing fields and conversation history to phrase the next follow-up.
> PROM or clinician-assigned questions are retrieved from MongoDB.

Important: this repository does not currently contain a question-management
admin screen or CRUD API.

## 6. Patient sends a text answer

```text
Text entered -> WebSocket text_input -> validate/sanitise -> LangGraph
-> extract form fields -> save MongoDB -> generate next question -> UI displays it
```

1. The frontend sends `text_input` from `useWebSocket.ts:384`.
2. The backend WebSocket receives the message in `server.py:1988`.
3. The interview graph is invoked.
4. `extract_form_data` extracts clinical facts from the conversation.
5. `validate_section` checks whether required information is missing.
6. `advance_section` moves forward when the current section is complete.
7. `generate_question` creates the next patient-friendly question.
8. `save_customer_info()` writes the updated attempt to MongoDB at
   `server.py:1219`.
9. The backend sends a `text_message` with the updated interview state.
10. `useWebSocket.ts:206` receives it and the React screen updates.

Graph definition: `DataHandling/src/graph/graph.py:59`.

Say to the client:

> Each answer is interpreted into structured form fields. The graph checks what
> is still missing, saves the current attempt, and returns the next relevant
> question with the updated progress.

## 7. Patient sends a voice answer

```text
Microphone -> audio_start -> audio chunks -> audio_end -> Gemini transcription
-> Google Cloud fallback if needed -> transcript returned -> normal answer flow
```

1. The frontend sends `audio_start` before recording at
   `TranscriptionInterface.tsx:1286`.
2. Audio chunks are sent through the existing WebSocket.
3. The frontend sends `audio_end` at `TranscriptionInterface.tsx:1356`.
4. The backend calls `transcribe_audio_bytes()` in
   `DataHandling/app/audio/stt.py:280`.
5. Gemini is attempted first at `stt.py:108`.
6. Empty or failed Gemini transcription falls back to Google Cloud STT at
   `stt.py:154`.
7. The backend returns a `transcription` WebSocket event.
8. The frontend receives it at `useWebSocket.ts:181` and places it in the answer
   box before it follows the normal interview-answer flow.

Say to the client:

> The browser's speech recognition is not used as the authoritative transcript.
> Audio is transcribed on the backend using Gemini, with Google Cloud STT as a
> fallback, and then processed exactly like a typed answer.

## 8. AI model flow

The local configuration currently uses:

| Job | Model |
|---|---|
| General conversation | `gemini-2.5-flash-lite` |
| Reasoning/form extraction | `gemini-2.5-flash` |
| Audio transcription | `gemini-2.5-flash` |
| Search/grounding | `gemini-2.5-flash` |
| Fallback | `gemini-2.5-flash` |

Model ownership and validation: `DataHandling/app/ai/models.py:21` and
`DataHandling/app/ai/models.py:69`.

Say to the client:

> Flash Lite handles lower-cost general work, while Flash handles reasoning,
> clinical extraction and audio. The model IDs are centrally configured and
> validated at startup instead of being scattered through the application.

Production model values must be confirmed from production environment
configuration. Never show the real `.env` file during screen sharing.

## 9. Database write and resume flow

```text
Interview state -> save_customer_info -> customer-info collection
-> key: userId + formId + attemptId -> later request loads same attempt
```

- MongoDB setup: `DataHandling/server.py:557`
- Save operation: `DataHandling/server.py:1219`
- Fetch user forms: `DataHandling/server.py:1379`
- Fetch one form: `DataHandling/server.py:1458`
- Attempt rules: `DataHandling/app/forms/attempts.py`
- Draft/in-progress/completed lifecycle: `DataHandling/app/forms/lifecycle.py`

Say to the client:

> `userId` identifies the patient, `formId` identifies the questionnaire, and
> `attemptId` identifies one completion. The attempt ID prevents a new intake from
> overwriting a patient's previous intake and allows an unfinished attempt to be
> resumed.

## 10. Report upload flow

```text
File selected -> authenticated REST upload -> ownership and file validation
-> S3 -> MongoDB job -> background worker -> document summary -> form updated
-> latest status returned to frontend
```

1. Frontend upload request: `TranscriptionInterface.tsx:2016`.
2. Backend endpoint: `DataHandling/server.py:4571`.
3. File policy checks: `DataHandling/app/uploads/policy.py`.
4. S3 integration: `DataHandling/upload/s3_client.py`.
5. Durable job logic: `DataHandling/app/jobs/reports.py`.
6. Document processing: `DataHandling/docscanner/service.py`.
7. Worker publishes the report result into the form at `server.py:601`.
8. The updated attachment/progress state is returned to the frontend.

Say to the client:

> Upload acknowledgement is separated from slower AI report processing. The file
> is validated and stored first, a durable MongoDB job processes it in the
> background, and the result is written only to the correct patient attempt.

## 11. Failure and fallback flow

| Failure | Behaviour |
|---|---|
| Missing/invalid token | Request is rejected; patient data is not returned |
| Consent missing | Interview start remains blocked |
| Gemini transcription fails | Google Cloud STT is attempted |
| Invalid upload | Upload is rejected before report processing |
| WebSocket disconnects | Frontend reconnect logic restores the connection |
| Database unavailable | Health/error response indicates the dependency problem |
| Report processing fails | Durable job is marked failed instead of silently disappearing |

Say to the client:

> The system has explicit failure boundaries. Authentication and consent fail
> closed, speech has a provider fallback, uploads are validated, and long report
> operations have durable job status.

## 12. Authentication responsibility

```text
Trusted invitation service creates JWT -> link contains #access_token=...
-> frontend consumes token -> backend validates signature, scope and patient
```

- Frontend token consumption: `customer-agent-frontend/src/config/auth.ts:3`
- REST Bearer header: `customer-agent-frontend/src/config/auth.ts:34`
- Backend JWT verification: `DataHandling/app/security/access_tokens.py`
- Local testing token tool: `DataHandling/scripts/generate_access_token.py`

Say to the client:

> The frontend never signs a trusted token. Production tokens must be issued by a
> trusted clinician or invitation service. The backend validates the signature,
> expiry, audience, scope and patient binding.

## 13. If an unexpected question is asked

Do not guess. Say:

> Let me trace the exact runtime path so I give you the implementation answer,
> not an assumption. The request enters here, then I will follow the service call,
> persistence and response path.

Then search in this order:

1. Frontend event or API call.
2. WebSocket message type or REST route.
3. Backend handler.
4. Service/AI/external-provider call.
5. MongoDB write or read.
6. Backend response event.
7. Frontend response handler and UI state.

That is a correct senior-engineer approach: trace first, then answer precisely.

## 14. Tabs to prepare before the meeting

1. `DataHandling/data/forms/FRM-01.md`
2. `DataHandling/src/prompts.py`
3. `DataHandling/src/graph/graph.py`
4. `DataHandling/app/ai/models.py`
5. `DataHandling/app/audio/stt.py`
6. `DataHandling/server.py` at the WebSocket endpoint
7. `customer-agent-frontend/src/hooks/useWebSocket.ts`
8. `customer-agent-frontend/src/components/TranscriptionInterface.tsx`
9. `customer-agent-frontend/src/config/auth.ts`
10. `DataHandling/app/security/access_tokens.py`

Do not open `.env`, credentials, private keys or production patient records while
sharing the screen.
