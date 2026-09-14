# Stance Configurable Agent Workflow Platform

## Production Architecture and Zero-Disruption Migration Plan

**Document status:** Architecture recommendation  
**Prepared:** 11 August 2026  
**Scope:** Customer Agent, Customer Agent Frontend, Clinician Agent, One View/VALD, Clinical Summarizer, Phase Analysis, and Audit Orchestrator

---

## 1. Executive summary

Stance currently operates several independently developed AI and data-processing systems:

- Customer Intake Agent
- Clinician Documentation Agent
- One View/VALD integration
- Clinical Summarizer
- Rehabilitation Phase Analysis
- Audit Orchestrator

These systems work, but each repository combines workflow orchestration, business behavior, prompts, infrastructure, persistence, protocols, and integrations. Consequently, adding a new agent generally requires copying or creating another backend, deployment configuration, WebSocket contract, model integration, and state-management implementation.

The required future architecture is a **configurable agent workflow platform** in which:

- A client creates workflows visually.
- Workflows are stored as versioned definitions.
- One durable runtime executes all workflow definitions.
- Existing capabilities are exposed as reusable, typed workflow nodes.
- New workflows using existing nodes require configuration rather than a new codebase.
- New code is required only when Stance introduces a genuinely new capability or external integration.

The safest migration strategy is not to merge or rewrite the existing repositories first. Stance should introduce a durable orchestration layer and compatibility gateway, register existing services as workflow activities, validate behavior in shadow mode, and then replace hardcoded internals node by node.

### Recommended platform direction

```text
Camunda 8 Self-Managed as the visual and durable orchestration layer
+ existing services initially registered as workflow activities
+ compatibility gateways preserving current APIs and WebSockets
+ LangGraph retained for selected AI conversation internals
+ MongoDB retained as the current clinical-domain data store
+ Redis retained for caching and short-lived operational state
+ centralized prompt, schema, tool, model, and policy registries
+ immutable workflow versions
+ shadow, canary, and instant rollback support
```

If Camunda licensing, BPMN, or operational requirements are unsuitable, the recommended alternative is Temporal with a custom React Flow workflow designer and a generic definition interpreter.

---

## 2. Assessment scope and verification boundary

The assessment covered approximately 70,000 source lines across the following repositories:

| Repository | Primary responsibility | Reviewed branch |
|---|---|---|
| `customerAgent/customer-agent` | Patient intake backend | `prod` |
| `customerAgent/customer-agent-frontend` | Patient intake frontend | `dev` |
| `Clinician-agent` | Clinician voice/text form assistant | `prod` |
| `One-view` | VALD synchronization and profile management | `v1` |
| `summarizer-phase-agent` | Clinical summary, phase analysis, and orchestration | `v27.5.3-prod` |

The review included:

- Runtime entry points
- HTTP and WebSocket routes
- State transitions
- AI model integrations
- MongoDB and Redis usage
- S3 and document processing
- VALD integrations
- Background jobs and concurrency
- Docker and deployment configuration
- Prompt and schema placement
- Test coverage
- Authentication and authorization boundaries visible in code
- Duplicate, legacy, and partially migrated implementations

This was a static repository assessment. It did not verify:

- The exact commits running in production
- Live MongoDB schemas, indexes, or volumes
- Production authentication handled outside these repositories
- Production latency and throughput
- Current cloud-account configuration
- Contractual or regulatory compliance status
- Vendor licensing and commercial terms

These items require production-environment validation before implementation.

---

## 3. Current system architecture

```text
                         STANCE PLATFORM
                               │
             ┌─────────────────┴──────────────────┐
             │                                    │
      Patient experience                  Clinician experience
             │                                    │
             ▼                                    ▼
   Customer Intake Agent                 Clinician Agent
   - Interviews patient                  - Transcribes notes
   - Fills intake form                   - Updates clinical form
   - Processes documents                 - Generates recommendations
             │                                    │
             └───────────────┬────────────────────┘
                             │
                      Clinical records
                             │
             ┌───────────────┴────────────────┐
             │                                │
             ▼                                ▼
     One View / VALD                  Summarizer + Phase Agent
     - Fetches VALD tests             - Builds patient timeline
     - Processes metrics              - Generates clinical summary
     - Caches results                 - Determines rehab phase
             │                                │
             └──────────── MongoDB ───────────┘
```

The systems are loosely coupled through the surrounding Stance platform, shared MongoDB collections, and service-to-service HTTP calls. There is no single durable runtime governing all executions.

---

## 4. Why the current implementation is not configurable

The hardcoding problem is not limited to prompts or form fields.

| Concern | Current implementation |
|---|---|
| Workflow order | Python branches, LangGraph edges, and background-task calls |
| Prompts | Large Python string constants |
| Forms | Python dictionaries and separate JSON fixtures |
| Model selection | Per-service model initialization and fallback code |
| Protocols | Different HTTP and WebSocket formats for each agent |
| Identity | Different user, patient, appointment, form, and VALD identifiers |
| State | Connection memory, MongoDB, Redis, files, and process memory |
| Job semantics | Different meanings for accepted, running, and completed |
| Authorization | Inconsistent or delegated outside the service |
| Deployment | Separate Compose files, ports, domains, and scripts |
| Observability | Different tracing and logging approaches |

A configuration file cannot safely replace infrastructure, authorization, state recovery, idempotency, or external integrations. The correct separation is:

```text
Agent = Versioned workflow configuration
      + Approved reusable capabilities
      + Runtime policies
```

Code should implement reusable capabilities. Configuration should decide how approved capabilities are combined.

---

## 5. Current repository findings

### 5.1 Customer Intake Agent

**Active production entry point:** `customerAgent/customer-agent/DataHandling/server.py`

Responsibilities include:

- FastAPI HTTP and WebSocket handling
- Per-connection interview state
- Faster-Whisper transcription
- Gemini conversation and structured extraction
- LangGraph interview execution
- MongoDB form persistence
- Consent handling
- S3 medical-document upload
- Bedrock document analysis
- ChromaDB retrieval
- Text-to-speech
- Legacy `HealthAgent` fallback

Important findings:

1. `server.py` remains the active production source of truth.
2. A newer `app/` package exists, but the main WebSocket is still implemented in the monolith.
3. The application contains both LangGraph and legacy-agent paths.
4. LangGraph has no durable checkpointer; full graph state is passed from connection state.
5. MongoDB stores form progress, but full workflow execution state is not durable.
6. Some form saves run in background tasks.
7. The frontend supports many dynamic question types, but the current backend does not consistently provide their metadata.
8. Form identity is effectively `userId + FRM-01`.
9. The frontend production build succeeds, but the current lint baseline reports 52 errors and 10 warnings.

The Customer Agent is the most complex and highest-risk workflow to migrate. It should be migrated after simpler background workflows have proven the platform.

### 5.2 Clinician Agent

**Active entry point:** `Clinician-agent/main.py`

Responsibilities include:

- WebSocket communication
- Base64 audio processing
- Google Speech-to-Text
- Transcript cleanup
- Gemini/Vertex form extraction
- Redis chat history, caching, rate limiting, and session state
- Form validation
- Background recommendations and suggestions
- Phoenix/OpenTelemetry tracing

Important findings:

1. It returns an updated form to its caller but does not own final clinical-record persistence.
2. The authentication payload is acknowledged but not validated in the service.
3. Its composite session identity concatenates `userId`, `appointmentId`, and `formKey` without a delimiter.
4. Full prompts may be exported to tracing, creating a patient-data exposure risk.
5. The repository contains large duplicated and historical implementations.
6. The active form pipeline currently sends the full transcription to `planTranscriptionSegment`, while objective, subjective, and RPE segments are empty.

The migration must initially preserve this behavior for parity. Improving segmentation must be a separate, evaluated behavioral change.

### 5.3 One View / VALD

**Active-looking entry point:** `One-view/update_vald_exercises_app.py`

Responsibilities include:

- Resolve `stanceId` to an associated VALD profile
- Fetch ForceDecks, ForceFrame, and Dynamo data
- Normalize VALD exercise metrics
- Calculate supported asymmetry and percentage change
- Create frontend graph structures
- Merge cached responses in MongoDB
- Batch processing
- VALD profile linking and unlinking

Important findings:

1. The active flow uses `associatedValdId[0]`.
2. A shared `lastRecordedUtc` watermark is applied across VALD products.
3. The online parser structures ForceDecks and ForceFrame but does not fully integrate Dynamo into its final parsed result.
4. Empty results can represent either no new data or some upstream errors.
5. The checked-in Docker Compose references modules not present in the repository.
6. A separate deployment script starts the current-looking entry point.
7. Linking VALD and updating MongoDB are separate operations without a distributed transaction.

VALD clients and parsers should remain reviewed code. They should be exposed as reusable workflow activities rather than rewritten as configuration.

### 5.4 Clinical Summarizer, Phase Analysis, and Audit Orchestrator

**Active entry points:**

- `summarizer-phase-agent/api/summary_api.py`
- `summarizer-phase-agent/api/phase_analysis_ws.py`
- `summarizer-phase-agent/orchestrator/main.py`

Responsibilities include:

- Watch MongoDB change streams
- Deduplicate report and VALD activity
- Apply generation thresholds
- Dispatch summary and phase jobs
- Build a chronological clinical timeline
- Generate an evidence-based clinical assessment
- Generate concise summary sections
- Generate rehabilitation-phase analysis
- Persist results to MongoDB

Important findings:

1. The downstream REST endpoints accept background jobs.
2. The orchestrator treats a successful HTTP acceptance as completed work.
3. A downstream AI job can fail after the queue has already been cleared.
4. Downstream job status is stored in process memory.
5. Summary and phase share one patient queue document.
6. Broad CORS and missing application authentication are visible in multiple APIs.
7. Several MongoDB connections allow invalid TLS certificates.
8. Credential artifacts appear tracked in Git and must be rotated.
9. There is no automated unit or integration test suite.
10. The deployment workflow selects the same Compose file for development and production branches.

This is the recommended first migration target because orchestration can be made durable while leaving existing AI-generation logic unchanged.

---

## 6. Current protocol differences

| System | Current input/output contract |
|---|---|
| Customer Agent | JSON control messages plus streamed binary audio |
| Clinician Agent | JSON containing text, current form, or base64 audio |
| One View | WebSocket request containing `stance_id` |
| Clinical Summarizer | REST background jobs plus colon-delimited WebSocket status strings |
| Phase Analysis | Separate REST and colon-delimited WebSocket status protocol |

These protocols must initially remain unchanged.

```text
Existing client
      ↓
Existing URL and payload
      ↓
Compatibility adapter
      ↓
Canonical workflow event
      ↓
Durable workflow runtime
```

The compatibility layer lets Stance centralize execution without forcing simultaneous frontend migrations.

---

## 7. Target architecture

```text
                     Workflow Control Plane
  ┌────────────────────────────────────────────────────────┐
  │ Visual Designer                                        │
  │ Workflow Registry                                      │
  │ Forms and Schema Registry                              │
  │ Prompt Registry                                        │
  │ Tool Registry                                          │
  │ Connection and Secret References                       │
  │ Draft → Review → Publish → Rollback                    │
  └──────────────────────────┬─────────────────────────────┘
                             │
                             ▼
                     Durable Workflow Engine
  ┌────────────────────────────────────────────────────────┐
  │ Version-pinned executions                              │
  │ Durable state, timers, and waits                       │
  │ Conditions, loops, and parallel branches               │
  │ Retries, timeouts, and incident handling               │
  │ Human approval                                         │
  │ Cancellation and compensation                          │
  └──────────────────────────┬─────────────────────────────┘
                             │
             ┌───────────────┼────────────────┐
             ▼               ▼                ▼
     Compatibility      Generic node     Existing legacy
       gateways            workers           services
             │               │                │
             └───────────────┼────────────────┘
                             ▼
             MongoDB / Redis / S3 / VALD / LLMs
```

### 7.1 Control plane

The control plane manages:

- Workflow drafts and immutable published versions
- Prompts and prompt versions
- Form and output schemas
- Models and fallback policies
- Tool definitions
- Connections and secret references
- Tool permissions
- Human-approval rules
- Retry and timeout policies
- Test datasets
- Environment promotion
- Rollback labels

The control plane does not execute patient workflows.

### 7.2 Execution plane

The runtime must support:

- Durable checkpoints
- Durable message waits
- Idempotency
- At-least-once delivery handling
- Per-patient concurrency control
- Retries with backoff
- Incident/dead-letter handling
- Human tasks
- Cancellation
- Workflow and node timeouts
- Compensation for partial external operations
- Execution history
- Audit events
- Version-pinned execution
- Structured input/output validation

### 7.3 Data plane

During migration:

- MongoDB remains the clinical-domain source of truth.
- Existing collection names remain unchanged.
- Redis remains a cache and short-lived operational store.
- Workflow state moves to the durable workflow engine.
- S3 remains the medical-document object store.
- Secrets move to a managed secret store.

The workflow engine must not become a duplicate clinical database.

---

## 8. Reusable workflow node catalogue

### 8.1 Trigger nodes

- HTTP request
- WebSocket event
- MongoDB change event
- Schedule
- File upload
- Manual trigger
- External webhook

### 8.2 Communication nodes

- Send WebSocket event
- Send HTTP response
- Wait for user message
- Stream token
- Stream audio
- Send notification

### 8.3 AI nodes

- LLM completion
- Structured extraction
- Intent classification
- Question generation
- Summary generation
- RAG retrieval
- Document analysis
- Tool-using agent
- Evaluation
- Guardrail

### 8.4 Data and clinical nodes

- Load patient
- Load appointment
- Load form
- Apply form patch
- Validate form
- Calculate progress
- Record consent
- Upload attachment
- Load clinical timeline
- Save clinical summary
- Save rehabilitation phase

### 8.5 VALD nodes

- Resolve VALD profile
- Fetch ForceDecks
- Fetch ForceFrame
- Fetch Dynamo
- Parse VALD metrics
- Merge cached response
- Update product watermark

### 8.6 Control-flow nodes

- Condition
- Switch
- Parallel branch
- Bounded loop
- Retry
- Wait
- Human approval
- Sub-workflow
- Error boundary
- Stop

---

## 9. Canonical event contract

Every external request should be converted into a canonical internal event:

```json
{
  "eventId": "uuid",
  "eventType": "conversation.message",
  "tenantId": "stance",
  "workflowKey": "customer-intake",
  "workflowVersion": 4,
  "executionId": "uuid",
  "sessionId": "uuid",
  "subject": {
    "type": "patient",
    "id": "stance-patient-id"
  },
  "actor": {
    "type": "patient",
    "id": "authenticated-user-id"
  },
  "idempotencyKey": "unique-request-key",
  "occurredAt": "ISO-8601",
  "payload": {},
  "trace": {
    "correlationId": "uuid"
  }
}
```

The platform must establish one canonical patient identity while retaining mappings for:

- MongoDB user ObjectId
- `stanceId`
- Appointment ID
- Form key and form ID
- VALD profile ID
- Existing composite clinician-session key

---

## 10. Node contract

Every executable node must be registered with a typed contract:

```json
{
  "type": "stance.form.extract",
  "version": "1.2.0",
  "inputSchema": {},
  "outputSchema": {},
  "permissions": [
    "patient.read",
    "form.write"
  ],
  "timeoutSeconds": 30,
  "retryPolicy": {
    "maximumAttempts": 3,
    "backoff": "exponential"
  },
  "idempotent": true
}
```

Every node execution records:

- Workflow and node version
- Input reference
- Output reference
- Start and completion timestamps
- Status
- Attempt number
- Error classification
- Actor and subject
- Correlation ID
- Model and prompt version when applicable
- Token and cost metadata when applicable

Sensitive input and output must be redacted from general logs.

---

## 11. Workflow-definition safety rules

Workflow configuration must not permit:

- Arbitrary Python or JavaScript
- Arbitrary shell commands
- Direct database credentials
- Unrestricted outbound URLs
- Unvalidated expressions
- Unbounded loops
- Unbounded parallelism
- Runtime editing of published versions
- Access to unapproved tools
- Model-selected authorization decisions

Use:

- JSON Schema for node inputs and outputs
- A safe expression language such as CEL or JSON Logic
- Secret references rather than secret values
- Allowlisted network destinations
- Role-based and attribute-based tool permissions
- Explicit maximum loop and concurrency values

Never execute configuration through Python `eval`.

---

## 12. Technology recommendation

### 12.1 Primary: Camunda 8 Self-Managed

Camunda best matches the requirement for a client-facing visual workflow platform with durable process execution.

Relevant capabilities include:

- Visual BPMN modeling
- Durable workflow state
- Timers and message correlation
- Human tasks
- Retries and incidents
- Workflow history and operational inspection
- Deterministic rules around AI steps
- Service connectors
- AI-agent orchestration

Official references:

- [Camunda agentic orchestration](https://docs.camunda.io/docs/components/agentic-orchestration/agentic-orchestration-overview/)
- [Camunda AI Agent connector](https://docs.camunda.io/docs/components/connectors/out-of-the-box-connectors/agentic-ai-aiagent/)

Existing Python services can initially remain unchanged and be invoked through service-task workers or secured HTTP connectors.

Before selection, validate:

- Licensing and total operating cost
- Self-managed infrastructure requirements
- Identity-provider integration
- Backup and disaster recovery
- Data residency
- Audit retention
- Connector security
- Team BPMN skills

### 12.2 Alternative: Temporal plus React Flow

Temporal provides durable workflow execution and recovery after process or network failure.

Official references:

- [Temporal documentation](https://docs.temporal.io/)
- [React Flow component](https://reactflow.dev/api-reference/react-flow)

This alternative requires Stance to build and own:

- The visual workflow designer
- Workflow JSON/DSL
- Graph interpreter
- Node configuration panels
- Definition validation
- Publishing and rollback
- Execution visualization

Choose this option when complete platform control is more important than minimizing platform-development effort.

### 12.3 LangGraph

LangGraph should remain available for AI-specific inner workflows, particularly the Customer Agent conversation.

Official reference:

- [LangGraph persistence](https://docs.langchain.com/oss/python/langgraph/persistence)

LangGraph alone does not provide the complete client-facing control plane, workflow publishing lifecycle, access control, schema registry, and compatibility gateway required by this project.

### 12.4 Dify and Langflow

Dify and Langflow are useful for proof-of-concept AI workflow authoring:

- [Dify Workflow Studio](https://www.dify.ai/workflows)
- [Langflow visual editor](https://docs.langflow.org/concepts-overview)

They should not become the clinical system of record without validating durable waits, distributed recovery, version migration, fine-grained authorization, audit retention, tenancy, data residency, and commercial-license requirements.

### 12.5 n8n

n8n is suitable for surrounding automation such as notifications, schedules, webhooks, and third-party integrations.

- [n8n AI agents](https://n8n.io/ai-agents/)

It is not recommended as the authoritative runtime for multi-turn clinical sessions.

---

## 13. Zero-disruption migration strategy

### Phase 0: Baseline and production validation

1. Confirm the exact production commit for every service.
2. Inventory all live domains, ports, routes, and consumers.
3. Capture current HTTP and WebSocket contracts.
4. Create OpenAPI and AsyncAPI definitions.
5. Capture sanitized golden conversations and outputs.
6. Record current MongoDB mutations.
7. Establish latency, throughput, error, and cost baselines.
8. Add correlation IDs across services.
9. Add feature flags by workflow, tenant, center, and user.
10. Rotate tracked credentials.
11. Enforce authentication before internal service access.

Do not change prompts, models, schemas, database documents, or business behavior during this phase.

### Phase 1: Compatibility gateway

Introduce a gateway while preserving existing public contracts:

```text
customeragent.stance.health/ws/{clientId}
agent.stance.health/ws
oneview.stance.health/ws
summarizer.stance.health/*
```

Initially, the gateway forwards traffic unchanged and emits canonical events in shadow mode.

No frontend or MongoDB migration is required.

### Phase 2: Register current systems as legacy workflow activities

Create activities such as:

```text
legacy.customer.process-turn
legacy.clinician.process-form
legacy.vald.synchronize
legacy.summary.generate
legacy.phase.generate
```

The workflow engine controls scheduling and status while existing services continue to provide their current behavior.

### Phase 3: Migrate summary orchestration first

```text
MongoDB change
    ↓
Deduplicate event
    ↓
Check generation threshold
    ↓
Start summary job
    ↓
Wait for actual terminal job completion
    ↓
Start phase job
    ↓
Wait for actual terminal job completion
    ↓
Mark workflow complete
```

The new runtime must not treat HTTP acceptance as job completion.

Run the new workflow without production writes first and compare it with the current orchestrator.

### Phase 4: Migrate Phase Analysis

- Give summary and phase independent statuses.
- Preserve the existing AI prompts and parsers.
- Introduce durable retries and incidents.
- Keep existing MongoDB output collections.

### Phase 5: Migrate One View orchestration

First reproduce existing behavior using current VALD clients and parsers as activities.

After parity, introduce separate product watermarks:

```json
{
  "watermarks": {
    "forcedecks": "...",
    "forceframe": "...",
    "dynamo": "..."
  }
}
```

The watermark change must not be combined with the initial orchestration cutover.

### Phase 6: Migrate the Clinician Agent

Preserve its current WebSocket input/output through an adapter:

```text
Legacy JSON/base64 message
      ↓
Compatibility adapter
      ↓
Transcribe
      ↓
Extract
      ↓
Validate
      ↓
Return current response format
```

Initially call the existing form processor. Replace the hardcoded segmentation only after parity is demonstrated.

### Phase 7: Migrate the Customer Agent

Migrate last because this workflow combines:

- Multi-turn conversation
- Binary audio streaming
- Form persistence
- Corrections
- Report upload
- Document processing
- Summary confirmation
- RAG questions
- TTS
- Reconnection
- LangGraph and legacy fallbacks

Every interview must be pinned to the workflow version with which it started.

```text
Session starts on customer-intake:v4
       ↓
v5 is published
       ↓
Existing session remains on v4
New sessions use v5
```

### Phase 8: Extract reusable activities

```text
Current service function
       ↓
Reusable activity implementation
       ↓
Old service delegates to activity
       ↓
Workflow calls activity directly
       ↓
Old service is retired after parity
```

Never simultaneously change orchestration, prompts, models, protocols, and database schemas.

---

## 14. Shadow, canary, and rollback model

Every workflow requires these operating modes:

```text
legacy_only
shadow
new_runtime_no_write
new_runtime_canary
new_runtime_primary
rollback
```

### Shadow mode

```text
Production request
    ├── Legacy service owns actual writes
    └── New workflow runs with writes disabled

Compare:
- Response
- Form patch
- Database mutation plan
- Workflow branch
- Classification
- Latency
- Errors
```

### Canary mode

Begin with:

- Internal test patients
- One selected center
- Explicit patient allowlists
- A small percentage of new sessions

### Single-writer rule

For every execution, exactly one runtime may own external writes.

- Shadow: legacy writes, new runtime is read-only.
- Canary: ownership is selected before execution starts.
- Never allow both runtimes to write the same patient workflow.

### Rollback

Rollback should require only a routing/feature-flag change:

```text
workflow.customer-intake = legacy
workflow.clinician-form = legacy
workflow.summary = legacy
```

Existing APIs and MongoDB schemas remain available during early migration phases, so rollback does not require reversing a data migration.

---

## 15. Testing strategy

### 15.1 Test layers

```text
Node unit tests
    ↓
Tool contract tests
    ↓
Workflow-definition validation
    ↓
Workflow simulation tests
    ↓
Legacy parity replay
    ↓
End-to-end API/WebSocket tests
    ↓
Load and failure-injection tests
```

### 15.2 Required test scenarios

- Patient and clinician identity isolation
- Concurrent messages for one session
- Reconnection and resume
- Duplicate input delivery
- Worker crash after external write
- Model timeout and provider fallback
- MongoDB outage
- Redis outage
- S3 failure
- VALD rate limit and partial device failure
- Summary accepted but downstream job fails
- Workflow cancellation
- Workflow-version publication during an active session
- Human approval timeout
- Rollback while executions are active
- Unauthorized tool access
- Invalid workflow definition
- Prompt or schema incompatibility

### 15.3 AI regression tests

For AI nodes, store sanitized datasets containing:

- Input conversation
- Existing form
- Expected allowed form patch
- Disallowed changes
- Expected intent
- Expected missing fields
- Required safety behavior

Evaluate:

- Field-level precision and recall
- Unsupported field mutation
- Hallucinated clinical facts
- Question repetition
- Correction accuracy
- Summary completeness
- Latency
- Token use
- Cost

---

## 16. Workflow publication gates

A workflow cannot be published unless it passes:

- Graph validation
- Input/output schema validation
- No orphan-node validation
- Bounded-loop validation
- Parallelism-limit validation
- Tool-permission validation
- Secret-reference validation
- Contract tests
- Golden replay tests
- AI regression evaluation
- Failure and retry tests
- Load test
- Cross-patient isolation test
- Audit-event validation
- Rollback test

Published workflow versions are immutable.

Lifecycle:

```text
draft
  ↓
validated
  ↓
reviewed
  ↓
staging
  ↓
canary
  ↓
active
  ↓
deprecated
```

---

## 17. Security and governance requirements

Before centralizing execution:

1. Rotate every credential or private key tracked in Git.
2. Remove secrets from repository history where required.
3. Use a managed secret store.
4. Require authentication for every external and internal request.
5. Authorize access by tenant, center, role, patient, workflow, and tool.
6. Prevent patient identifiers from functioning as authorization tokens.
7. Restrict CORS to approved origins.
8. Require valid TLS certificates.
9. Encrypt workflow state and sensitive data at rest and in transit.
10. Redact clinical content from standard logs and traces.
11. Store immutable audit records for workflow publication and execution.
12. Require human review for clinically consequential outputs.
13. Restrict workflow authors from assigning tools they are not permitted to use.
14. Add retention and deletion policies for execution state and prompts.

The LLM must not make authentication, authorization, or final clinical-validation decisions.

---

## 18. Observability requirements

The platform needs one trace across:

```text
Frontend request
    ↓
Compatibility gateway
    ↓
Workflow execution
    ↓
Node/activity
    ↓
Model or integration call
    ↓
Database mutation
```

Required metrics:

- Workflow starts and completions
- Active and waiting executions
- Node duration
- Retry count
- Incident count
- Dead-letter count
- Model latency and errors
- Tokens and estimated cost
- Tool-call latency and errors
- WebSocket connections and reconnects
- Form extraction accuracy
- Shadow parity rate
- Canary error rate
- Patient-isolation violations

Prompt and trace platforms must be configured to avoid exposing identifiable clinical data.

---

## 19. Deployment model

Recommended environments:

```text
development
staging
production
```

Required separation:

- Separate workflow deployment labels
- Separate credentials
- Separate databases or strongly isolated schemas
- Separate object-storage prefixes/buckets
- Separate observability environments
- No production patient data in development

Recommended deployment practices:

- Immutable container images
- No production source-code bind mounts
- Dependency pinning
- Software bill of materials
- Vulnerability scanning
- Database migrations as versioned jobs
- Rolling or blue/green deployment
- Readiness and liveness probes
- Backup and restore testing
- Disaster-recovery runbook

---

## 20. Delivery sequence

### Milestone 1: Foundations

- Confirm production baselines
- Security remediation
- Contract inventory
- Canonical identity and event contracts
- Feature-flag infrastructure
- Workflow-engine proof of concept

### Milestone 2: Orchestration without behavior changes

- Compatibility gateway
- Legacy service activities
- Durable job identifiers
- Summary and phase shadow workflows
- Operational dashboards

### Milestone 3: First production workflows

- Summary workflow canary
- Phase workflow canary
- Completion acknowledgement correction
- Rollback validation

### Milestone 4: Integration workflows

- One View workflow
- Per-device watermark design
- Profile-link compensation/reconciliation

### Milestone 5: Interactive agents

- Clinician workflow
- Customer workflow
- Durable user-message waits
- Version-pinned sessions
- Audio compatibility adapters

### Milestone 6: Consolidation

- Extract shared activities
- Retire duplicated code
- Retire unused deployments
- Unify dependency management
- Archive old repositories only after parity and retention requirements are met

---

## 21. Explicit non-goals during initial migration

The initial orchestration migration must not also attempt to:

- Redesign clinical forms
- Change prompts
- Change LLM providers
- Correct every existing behavioral defect
- Rename MongoDB collections
- Replace all frontends
- Rewrite VALD parsers
- Merge all repositories
- Introduce autonomous clinical decisions

These are separate projects that should follow after platform stability.

---

## 22. Architecture decision

### Decision

Adopt a durable visual workflow orchestration layer and migrate existing services through compatibility activities before extracting their internal capabilities.

### Rationale

- Meets the client requirement for configurable workflows.
- Avoids a new codebase per agent.
- Preserves current production behavior.
- Provides safe incremental migration.
- Enables rollback without data reversal.
- Separates workflow orchestration from capability implementation.
- Introduces durable job and session state.
- Supports human review and auditability.

### Consequences

- The existing repositories remain temporarily operational.
- Stance must maintain compatibility adapters during migration.
- Node contracts and workflow governance become platform responsibilities.
- The team must learn BPMN/Camunda or build and maintain a custom workflow DSL and designer.
- New workflows become configuration-only when their required capabilities already exist.

---

## 23. Final conclusion

The current architecture is functionally valid for the existing agents but is not scalable for the client’s configurable-workflow requirement.

The production-safe target is:

```text
One visual workflow control plane
+ one durable execution engine
+ one versioned workflow registry
+ one reusable node/tool catalogue
+ compatibility gateways for existing clients
+ current services retained initially as activities
+ gradual capability extraction
+ immutable versions, shadow runs, canaries, and rollback
```

Stance should not begin by combining repositories. It should first separate orchestration from implementation. Once current services are safely callable as workflow activities, each hardcoded behavior can be moved into reusable nodes without affecting working production flows.

This changes future development from:

```text
New agent → new repository → new server → new deployment
```

to:

```text
New agent using existing capabilities → create and publish workflow
New capability → implement one reusable node → use it in any workflow
```
