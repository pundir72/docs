# Clinician Agent — 20-minute project guide

## Project goal

Clinician Agent helps clinicians create structured patient reports through
manual entry, typed notes, or voice recordings.

The product has two codebases:

- **Frontend:** displays patients, appointments and forms; records audio; shows
  AI changes; and saves approved reports.
- **AI backend:** converts audio to text and uses Gemini to turn clinical notes
  into structured form updates.

The existing Stance GraphQL API is a separate external system. It supplies
patient and appointment data and persists final reports.

Client-friendly description:

> The clinician selects a patient and appointment, speaks or types clinical
> notes, reviews the fields filled by AI, and saves the approved report to the
> existing Stance platform.

## Main architecture

~~~text
Clinician
   |
React frontend
   |---- GraphQL over HTTPS ----> Stance API ----> report database
   |
   +---- WebSocket /ws ---------> Python/FastAPI AI backend
                                      |
                                      +--> Google Speech-to-Text
                                      +--> Vertex AI / Gemini
                                      +--> Redis
                                      +--> Phoenix tracing
~~~

The browser has two independent connections:

1. **GraphQL:** reads centers, patients, appointments and reports, then saves the
   final approved form.
2. **WebSocket:** sends audio or text plus the current form to the AI backend,
   then receives transcription, updated form data and suggestions.

This distinction is important while debugging. AI processing can succeed while
GraphQL saving fails, or GraphQL can work while the WebSocket is offline.

## Complete runtime flow

### 1. Select patient and appointment

The home page loads centers, patients and appointments from GraphQL. The
clinician selects a form and starts the session.

Main files:

- clinician-agent-FE-v2/src/pages/Index.tsx — selection and navigation.
- src/hooks/use-centers.tsx — loads centers.
- src/hooks/use-patients.tsx — searches patients.
- src/hooks/use-appointments.tsx — loads appointments.
- src/utils/graphql-client.ts — GraphQL transport and request headers.

### 2. Open and initialize the form

The URL includes the form type, patient ID and appointment ID. The frontend
loads the matching schema and looks for an existing report. Existing First
Assessment values are restored. A newly initialized assessment can prefill
values from the appointment's Report.records.

Main files:

- src/App.tsx — application routes.
- src/pages/form-page.tsx — joins the form, recorder and save actions.
- src/hooks/use-form-management.tsx — initialization, report ID, reset, manual
  save and stale-request protection.
- src/schemas/form-schemas.ts — frontend form templates.
- src/utils/first-assessment.ts — converts saved API data to renderer data.
- src/utils/records-to-form.ts — maps clinical records into form fields.

### 3. Enter typed notes or record voice

The browser requests microphone permission and records audio. Typed notes and
audio are sent through the same WebSocket connection with the current form.

Main files:

- src/components/audio/voice-recorder-section.tsx — visible recorder area.
- src/components/audio/audio-recorder.tsx — captures browser audio.
- src/hooks/use-microphone-permission.tsx — permission status.
- src/hooks/use-voice-recorder.tsx — coordinates transcript and AI updates.
- src/hooks/use-web-socket.ts — connection, request IDs, timeouts and responses.

Microphone access requires HTTPS or localhost. Browsers normally block it on a
plain HTTP public IP.

### 4. Backend transcribes and processes the note

The FastAPI /ws endpoint validates the message and applies admission controls.
Audio goes to Google Speech-to-Text. The transcript and current form then go to
Gemini. The response is normalized and validated before being returned.

Main files:

- Clinician-agent/main.py — backend entry, WebSocket, audio and health routes.
- src/llm/functionalities.py — processing orchestration.
- src/llm/llm_manager.py — Gemini clients, routing and usage.
- src/prompts.py — clinical text-to-field instructions.
- src/forms/registry.py — form strategies and section defaults.
- src/schemas/validation.py — output and numeric validation.
- src/runtime.py — bounded workers, timeouts and follow-up processing.

First Assessment uses two parallel AI groups:

~~~text
Group 1: clinical details + subjective findings + objective findings
Group 2: goals + recommendations + patient advice
~~~

This reduces model calls while keeping each prompt focused.

### 5. Frontend applies the AI result

The WebSocket returns structured form data. The renderer merges valid changes,
highlights updates and protects manual edits made while AI was processing.

Main files:

- src/components/forms/form-renderer.tsx — generic form renderer.
- src/hooks/use-form-renderer.ts — renderer state.
- src/handlers/form-llm-update-handler.ts — applies AI responses.
- src/reducers/form-renderer.reducer.ts — field and array state changes.

The clinician must review generated content before saving it.

### 6. Save through GraphQL

Manual and automatic saves share one payload builder and save transport. First
Assessment changes its frontend objectiveAssessment object into the API's
objectiveAssessments list.

The serializer also converts recognized AI aliases:

~~~text
name        -> testName
details     -> comments
description -> conclusion
~~~

Unsupported fields are removed before GraphQL validation. Saves for the same
appointment are ordered in the browser.

Main files:

- src/utils/form-submission.ts — common submission and feedback.
- src/utils/report-payload.ts — strict GraphQL payload conversion.
- src/utils/graphql-client.ts — mutation and save confirmation.
- src/utils/auto-submit-manager.ts — delayed automatic saves.

The external GraphQL resolver performs the final database write. Its backend
implementation is outside this workspace.

## Supported forms

| Form | Purpose | AI route |
|---|---|---|
| firstAssessment | History, complaints, findings, goals and advice | Two grouped calls |
| assessment | Follow-up plan, subjective note, tests and RPE | Registered processors |
| snc | Strength and conditioning plan | Legacy-compatible route |
| physio | Physiotherapy tests | Legacy-compatible route |

Frontend structures are in src/schemas/form-schemas.ts. Backend routing is in
Clinician-agent/src/forms/registry.py.

## Data and service ownership

| System | Responsibility |
|---|---|
| Stance GraphQL API | Patient, appointment and report read/write operations |
| GraphQL report database | Final saved clinician report |
| FastAPI backend | Speech and AI request processing |
| Google Speech-to-Text | Audio transcription |
| Vertex AI / Gemini | Notes-to-form transformation |
| Redis | Temporary state, history, cache and rate limiting |
| Phoenix/OpenTelemetry | AI traces and diagnostics |
| Browser localStorage | Selected IDs and temporary local state |

Frontend build variables:

- VITE_API_URL and VITE_API_KEY configure GraphQL.
- VITE_WS_URL and VITE_WS_API_KEY configure the AI WebSocket.

VITE variables are visible in the browser bundle. They must not replace real
clinician/session authorization.

## Deployment overview

The frontend Docker build uses Node to compile Vite and Nginx to serve only the
static result:

- clinician-agent-FE-v2/Dockerfile
- clinician-agent-FE-v2/docker-compose-dev.yml
- clinician-agent-FE-v2/nginx.conf
- clinician-agent-FE-v2/UAT_DOCKER_DEPLOYMENT.md

The backend Compose file starts FastAPI, Redis and optional Phoenix:

- Clinician-agent/Dockerfile
- Clinician-agent/docker-compose-dev.yml

Expected deployment checks:

~~~text
Frontend /health -> HTTP 200 and "ok"
Backend  /health -> HTTP 200 and status "ok"
WebSocket /ws    -> HTTP 101 Switching Protocols
~~~

## Five-minute client demo

1. Select a patient, appointment and First Assessment.
2. Explain that GraphQL loads existing report data.
3. Enter or speak a clinical note.
4. Show the WebSocket request and explain Speech-to-Text plus Gemini.
5. Review the highlighted changes.
6. Correct one measurement and clear one finding.
7. Save the assessment.
8. Refresh and demonstrate that GraphQL reloads the saved data.
9. Show frontend and backend health checks.

Suggested demo note:

> The patient reports right knee pain when descending stairs. Right knee
> flexion is 125 degrees with mild medial joint-line tenderness. Correct right
> knee flexion to 130 degrees. The patient reports no knee instability. Advise
> avoiding high-impact activity until review.

Expected demo outcome:

- The value is stored as right knee 130 degrees without a duplicate general
  value.
- “No instability” remains a negative finding.
- A current measurement does not become a goal.
- Advice appears in patient advice.
- Saved fields reload after refresh.

## Client question to code-file map

| Client question | Open this file |
|---|---|
| Where are pages and routes? | frontend src/App.tsx |
| How is the form page assembled? | frontend src/pages/form-page.tsx |
| Where are form fields defined? | frontend src/schemas/form-schemas.ts |
| How is voice recorded? | frontend src/components/audio/audio-recorder.tsx |
| How does the browser contact AI? | frontend src/hooks/use-web-socket.ts |
| How are reports initialized and reset? | frontend src/hooks/use-form-management.tsx |
| How is the GraphQL payload validated? | frontend src/utils/report-payload.ts |
| Where does the backend start? | backend main.py |
| How is each form routed? | backend src/forms/registry.py |
| Where are AI rules written? | backend src/prompts.py |
| Where is Gemini called? | backend src/llm/llm_manager.py |
| Where is output validated? | backend src/schemas/validation.py |
| Where is old code retained? | Each repository's legacy directory |

## Key implementations completed

- Preserved replaced historical code in legacy directories.
- Added registry-based form routing and grouped First Assessment processing.
- Reduced First Assessment processing to two parallel model calls.
- Added bounded workers, timeouts, rate admission and request IDs.
- Added safer model clients and provider fallback behavior.
- Added numeric, blank, zero, RPE and load validation.
- Prevented duplicated general and side measurement values.
- Prevented measurements from being invented as clinical goals.
- Added structured, evidence-checked clarification suggestions.
- Protected manual edits from late AI responses.
- Fixed reset for prefetched, renderer and voice state.
- Unified manual and automatic GraphQL saves.
- Added ordered saves and confirmed-save handling.
- Added strict GraphQL field normalization and alias mapping.
- Added existing-report reload and Report.records prefill.
- Removed active hardcoded frontend key fallbacks.
- Added frontend Docker/Nginx packaging, SPA routing and health checks.
- Added regression tests for the main corrected flows.

## Current limits

The project is suitable for controlled UAT after environment and access controls
are configured. Do not describe it as completely production-ready yet.

- Gateway or session authorization must protect WebSocket access.
- Previously committed keys should be rotated.
- Normal microphone access requires HTTPS/WSS.
- Cross-client overwrite protection needs server-side version enforcement.
- AI output still requires clinician review and broader clinical evaluation.
- Legacy form paths do not have the same validation depth as registered forms.

## One-sentence summary

Clinician Agent is a React clinical-report interface that uses FastAPI,
Speech-to-Text and Gemini to turn voice or typed notes into reviewable
structured forms, then saves clinician-approved reports through Stance's
existing GraphQL platform.
