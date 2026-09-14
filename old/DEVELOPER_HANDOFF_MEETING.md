# OneView–VALD Pull Request Review Guide

> This is a local preparation document. It is ignored by Git and is not included
> in the pull request.

## Meeting objective

Review the cumulative local implementation, confirm assumptions with the previous
developer, agree on any required corrections, and approve the sequence for commit,
staging verification, migration canary, and production rollout.

## Opening summary

The work was intentionally completed in this order:

1. Protect synchronization correctness and existing patient history.
2. Make VALD and MongoDB access reliable and bounded.
3. Add security, observability, tests, and repeatable operations.
4. Add controlled migration and rollback-safe production deployment.

The highest-risk original behaviours were shared device watermarks, whole-cache
replacement, hidden fetch failures reported as “no data,” concurrent processing of
the same patient, and deployment that removed the current exercise container before
validating its replacement.

Local implementation status is 38/38 planned tasks. The suite contains 226 tests and
currently passes together with Ruff, compilation, secret scanning, shell syntax,
Git whitespace checks, parser-asset validation, and Docker Compose configuration.

This does **not** mean production deployment is complete. No production index was
created, no schema migration was applied, and no production container was replaced.
The approved patient ID `683da5b2e2467bc63941a172` was used for local candidate
integration testing against the configured data/VALD services: historical fetch,
history preservation, persistence, watermarks, immediate no-data replay, liveness,
and readiness passed. No other patient was used for a live synchronization test.

## Why the pull request is large

The PR is based on production release `22c947f` and currently contains 122 changed
paths: 41 test files, 39 application/source files, 8 deployment files, 7 parser
asset files, 4 operational Markdown files, and root configuration/entry-point
changes. The reported line count is inflated by 4,754 CSV data rows and the newly
added regression suite; it is not 17,000 lines of runtime business logic.

The changes should be reviewed in this order:

1. Data correctness and compatibility.
2. VALD/Mongo reliability and bounded concurrency.
3. Security and observability.
4. Batch processing and controlled migration.
5. VALD-only deployment and rollback.
6. Tests and versioned parser assets.

Personal notes, meeting questions, ad-hoc debug scripts, patient Excel exports,
and unrelated main/updater deployment definitions were removed from the PR.

## Resulting architecture

```text
WebSocket client / scheduled batch
              |
              v
Validated association + per-patient distributed lease
              |
              v
Bounded synchronization worker
              |
       +------+------+
       |             |
       v             v
 ForceDeck        ForceFrame
 paginated,      paginated,
 retry-bounded   retry-bounded
       |             |
       +------+------+
              v
    Versioned parser response
              v
History merge by source test ID
              v
MongoDB vald-exercise-data
  - reads: MONGO_READ_URI
  - writes: MONGO_URI
  - independent product watermarks
  - schema and document-size validation
```

Dynamo code remains available for future work, but Dynamo is not called and does
not affect synchronization decisions or watermarks in the current product scope.
Existing cached Dynamo data is preserved.

## Parser CSV and image files

These seven files appear as additions because the target Git release did not track
them, although the checked production/server working directory contained
`metric-pool/` and `graph-pool/` and the parser imports the CSV mappings directly.

| Asset | Purpose | Runtime use |
|---|---|---|
| `Graph pool stance - Sheet3.csv` | Primary exercise/metric/single-or-multi to graph-type mapping | Loaded first by the parser |
| `Graph pool stance - missing metrics.csv` | Fills mappings absent from the primary sheet without overwriting it | Loaded second by the parser |
| `Graph pool stance - all the new metrics.csv` | Identifies known exercise/metric pairs so an unmapped pair receives a deterministic graph default | Parser reads only `Exercise Name` and `Metric Name` |
| `graph-pool/1.png` to `3.png` | Reference screenshots defining the graph-name/design catalogue represented by the CSV graph types | Not returned by the API; checksum and dimensions are validated as versioned reference assets |
| `config/parser-assets.json` | Declares version, expected hashes, schemas, row minimums, and PNG dimensions | Validated by CI, Docker build, startup, and readiness |

The PNGs contain generic graph-design examples and no patient names or medical
records. One source CSV includes a `Stance IDs` provenance column; the parser does
not read or return that column. If the repository has a policy prohibiting opaque
patient identifiers in configuration assets, agree with the reviewer to sanitize
that unused column in a separate reviewed change rather than pretending it is used.

Concise answer if asked why these files are new: “They were unversioned production
parser inputs/reference material. Tracking and hashing them makes a release
reproducible and makes accidental mapping drift fail visibly.”

## Completed task register

### Phase 1 — Synchronization correctness and data safety

| # | Task | Why it was required | What was implemented | Main evidence |
|---:|---|---|---|---|
| 1 | Secret management | Tracked fallback credentials could expose production access and hide missing configuration. | Removed embedded credentials, added strict required-environment loading, safe `.env.example`, ignore/build exclusions, and a non-printing secret scanner. | `src/utils/env_config.py`, `src/utils/secret_scanner.py`, `scan_repository_secrets.py`, `tests/test_secret_scanner.py` |
| 2 | MongoDB read/write separation | Reads and mutations previously used the same connection setting. | `MONGO_READ_URI` now serves document/watermark reads; `MONGO_URI` serves persistence, leases, indexes, and migration writes. Both are validated. | `src/file_handling/vald_exercise_db.py`, `tests/test_mongo_connection_routing.py` |
| 3 | Current product scope | Dynamo was fetched even though it is future scope and could incorrectly influence current sync decisions. | Removed Dynamo from the active fetch, parser, response decision, and watermark path while preserving already cached Dynamo fields. | `src/file_handling/data_updater.py`, `tests/test_dynamo_scope.py` |
| 4 | Independent watermarks | One shared timestamp allowed newer ForceDeck data to skip older unseen ForceFrame data, or vice versa. | Added `syncState.forceDeck` and `syncState.forceFrame` watermarks with temporary read fallback to legacy `lastRecordedUtc`. | `src/file_handling/vald_exercise_db.py`, `tests/test_independent_watermarks.py` |
| 5 | Persistence ordering | A watermark could advance even when the exercise-data write failed. | Added an ordered persistence operation: store/merge data first, then advance only successful product watermarks; failures remain failures. | `persist_sync_result()`, `tests/test_persistence_ordering.py` |
| 6 | Cross-product preservation | A response containing one product could replace the complete cached object and erase the other product. | Writes now update only returned non-empty exercise paths and preserve other products, flags, unrelated fields, and cached Dynamo. | `push_exercise_data()`, `tests/test_cross_product_preservation.py` |
| 7 | Historical session preservation | Incremental parsing could replace earlier sessions and replay could create duplicates. | Added history merging and deduplication using internal stable `_sourceTestIds`, preserving earlier sessions and deterministic order. | `src/file_handling/exercise_history.py`, `tests/test_historical_session_preservation.py` |
| 8 | Timestamp precision | A one-second tolerance could reject legitimately newer subsecond records. | UTC parsing now preserves microseconds and offsets; replay safety comes from stable test IDs rather than timestamp tolerance. | `latest_fetch.py`, `tests/test_timestamp_precision.py` |
| 9 | Failure versus no data | Authentication, HTTP, and parsing failures could become an empty object and appear successful. | Introduced explicit `new_data`, `no_data`, and `failed` outcomes with safe error codes and separate retry decisions. | `FetchResult`, `FetchStatus`, `tests/test_fetch_outcomes.py` |
| 10 | Patient-to-VALD association | `associatedValdId` is an array, but silently choosing its first item could sync the wrong person. | Added deterministic validation: exactly one non-blank unique association is accepted; zero or multiple mappings fail explicitly. | `src/utils/vald_association.py`, `tests/test_vald_association.py` |
| 11 | Cached API contract | Responses used inconsistent string booleans and stored sessions lacked source traceability. | Added Pydantic response schema version `2.0`, real booleans/status/error fields, and internal test-ID traceability removed from public responses where required. | `src/models/sync_response.py`, `tests/test_sync_response_contract.py` |

### Phase 2 — Reliability, performance, and concurrency

| # | Task | Why it was required | What was implemented | Main evidence |
|---:|---|---|---|---|
| 12 | VALD pagination | One response page could omit tests when VALD returned multiple pages. | Added shared `ModifiedFromUtc` pagination, duplicate-test protection, repeated-cursor detection, and final-page/204 handling for ForceDeck and ForceFrame. | `src/utils/vald_pagination.py`, `tests/test_vald_pagination.py` |
| 13 | HTTP reliability | Some requests had no timeout and retry behaviour differed between endpoints. | Added bounded connect/read timeouts, transient-only retries, exponential backoff/jitter, and `Retry-After` handling. | `src/utils/http_client.py`, `tests/test_http_reliability.py` |
| 14 | VALD configuration | Tenant and Australia-East URLs were hardcoded across modules. | Added required tenant UUID and approved region selection (`aue`, `use`, `euw`) with centralized endpoint construction and startup validation. | `src/utils/vald_config.py`, `tests/test_vald_configuration.py` |
| 15 | OAuth token safety | Concurrent requests could refresh the same token repeatedly and token calls could stall. | Kept expiry-aware caching, added timeout/retry handling and synchronized refresh so concurrent callers reuse one token. | `vald_fetch_utils.py`, `tests/test_oauth_token_safety.py` |
| 16 | Async request handling | Blocking PyMongo/requests work inside the async WebSocket could freeze unrelated requests. | Moved synchronization into a bounded executor while retaining async health/WebSocket responsiveness and controlled shutdown. | `src/utils/sync_executor.py`, `tests/test_async_sync_execution.py` |
| 17 | Request amplification | Each patient could require a tenant-wide profile download and sequential details per test. | Replaced tenant-wide profile listing with direct profile lookup and added bounded parallel ForceDeck trial/ForceFrame metric retrieval. | `fetch_forcedeck.py`, `fetch_forceframe.py`, `src/utils/bounded_concurrency.py`, `tests/test_request_amplification.py` |
| 18 | MongoDB connections | Opening/closing a client for each operation increased latency and connection churn. | Added application-lifecycle read/write clients, configurable pools/timeouts, clean shutdown, and readiness reuse. | `src/utils/mongo_config.py`, `tests/test_mongo_client_settings.py`, `tests/test_mongo_connection_routing.py` |
| 19 | Database schema and indexes | Runtime assumptions about `stanceId`, associations, product shape, and uniqueness were unenforced. | Added application-level Pydantic validation plus audit-first unique `stanceId` and association index management. Index creation is never automatic. | `src/models/vald_exercise_document.py`, `manage_vald_indexes.py`, `tests/test_vald_document_schema.py`, `tests/test_vald_mongo_indexes.py` |
| 20 | MongoDB document growth | A patient document could grow toward MongoDB’s 16 MiB hard limit without warning. | Added BSON projected-size enforcement, configurable warning/max thresholds, read-only collection audit, and targeted field updates instead of whole-document replacement. | `src/utils/document_growth.py`, `audit_vald_document_sizes.py`, `tests/test_document_growth.py` |
| 21 | Concurrent synchronization | WebSocket and batch workers could update the same patient concurrently and overwrite history/watermarks. | Added a MongoDB-backed per-patient lease with ownership, expiry, renewal, clean release, and monotonic watermark updates. | `src/utils/sync_lease.py`, `tests/test_sync_lease.py` |
| 22 | Batch retry and resume | Failed records were treated as completed and skipped forever on restart. | Versioned progress now separates success, no-data, retryable failure, and terminal failure; retries are bounded and reconciliation is explicit. | `src/utils/batch_progress.py`, `tests/test_batch_progress.py` |
| 23 | Batch concurrency and API limits | Six fixed workers could exceed tenant limits and could not be tuned without editing code. | Made workers/retry settings configurable and added a shared cross-process request limiter with backpressure. | `src/utils/shared_rate_limiter.py`, `tests/test_shared_rate_limiter.py` |
| 24 | Batch partition consistency | Workers independently sliced a changing unsorted cursor, risking duplicate or missing patients. | Added one immutable, checksummed, deterministically sorted manifest and validated fixed worker chunks. | `src/utils/batch_manifest.py`, `tests/test_batch_manifest.py` |

### Phase 3 — Security, operations, deployment, and maintainability

| # | Task | Why it was required | What was implemented | Main evidence |
|---:|---|---|---|---|
| 25 | Production endpoint security | Operational routes could expose patient diagnostics or stop jobs anonymously. | Protected metrics/debug/inspection/cron/batch-control routes with a strong operations key and audit-oriented logging; anonymous access fails closed. | `src/utils/operational_auth.py`, `tests/test_operational_auth.py` |
| 26 | WebSocket and CORS security | Wildcard credentialed CORS and unauthenticated/unbounded WebSocket messages exposed the processor. | Added explicit origin allow-listing, API-key authentication, strict request model, size limit, receive timeout, sliding-window rate limit, and bounded client tracking. | `src/utils/websocket_security.py`, `tests/test_websocket_security.py` |
| 27 | Health and readiness | A static health response could report healthy with missing configuration or unavailable MongoDB. | Split liveness and readiness; readiness validates config, parser assets, runtime paths, and both MongoDB roles with a bounded timeout. | `src/utils/readiness.py`, `src/utils/container_healthcheck.py`, `tests/test_readiness.py`, `tests/test_container_runtime.py` |
| 28 | Logging and observability | Free-form logs and swallowed failures made a patient sync difficult to trace. | Added structured events, correlation IDs, per-product duration/count/failure metrics, synchronization outcomes, and safe error categories. | `src/utils/observability.py`, `src/utils/logging_config.py`, `tests/test_observability.py` |
| 29 | Docker deployment definition | The image default referenced missing `app:app`, and Compose declared main/updater applications absent from this checkout. | Made the repository explicitly VALD-only. The image, Compose, startup validation, and rollout all start `update_vald_exercises_app:app`; the service remains hardened with persistent runtime volumes and readiness checks. | `Dockerfile`, `docker-compose.yml`, `src/utils/vald_exercise_startup.py`, `docs/11-deployment.md` |
| 30 | Deployment port allocation | Historical scripts mixed ports belonging to separate applications. | Kept only the authoritative VALD port `8091` and loopback-only candidate port `8093`; ownership is checked before replacement. Main OneView and match-updater deployments remain in their separate server release paths. | `deployment/ports.env`, `deployment/port_utils.sh`, `tests/test_deployment_ports.py` |
| 31 | Parser configuration assets | Production parsing depends on CSV mapping files that previously existed as server-side files rather than versioned release inputs. Missing or modified files could silently change graph types. | Added three CSV mappings and three reference PNGs from the production/server code snapshot plus a versioned manifest that validates paths, hashes, CSV schemas/row counts, and PNG dimensions during CI, image build, startup, and readiness. | `metric-pool/`, `graph-pool/`, `config/parser-assets.json`, `src/utils/parser_assets.py`, `tests/test_parser_assets.py` |
| 32 | Metric units | ForceFrame impulse and repetition values could receive incorrect generic units. | Restored explicit `N·s` and `reps` mappings and regression coverage. | `src/parsers/vald_api_parser.py`, `tests/test_metric_units.py` |
| 33 | Automated tests and CI | The repository lacked repeatable protection against the identified data-loss and failure paths. | Added the current 226-test suite and GitHub CI for secrets, Ruff, dependency audit, compilation, parser assets, shell/Compose validation, tests, hardened image checks, and vulnerability scanning. | `tests/`, `.github/workflows/ci.yml`, `requirements-dev.txt` |
| 34 | Container hardening | The runtime image ran as root and included build tools. | Added a multi-stage image, pinned audited runtime dependencies, UID/GID `10001`, read-only exercise filesystem, dropped capabilities, no privilege escalation, health check, and image scanning. | `Dockerfile`, `requirements.txt`, `tests/test_container_hardening.py` |
| 35 | Scheduler/runtime consistency | Cron referenced a virtual environment that the exercise deployment did not install, and infrastructure values were duplicated. | Scheduler now executes batch code inside the deployed exercise container, uses a host lock, centralized infrastructure config, configurable schedule, and bounded log retention. | `deployment/run_scheduled_batch.sh`, `deployment/config.sh`, `setup_ec2_cron.sh`, `tests/test_scheduler_runtime.py` |
| 36 | Legacy and duplicate code | Multiple Mongo helpers, file-based fetch runners, hardcoded debug scripts, and shallow merge functions contradicted the active path. | Removed the old `matches` writer, file-based `fetchvald.py`, obsolete `/home/ubuntu/scrapper` launcher, shared-watermark writer, shallow merges, and unsafe direct reset route. Main/updater deployment scripts were also removed because their entry points are not in this VALD-only repository. | `tests/test_legacy_consolidation.py`, `tests/test_deployment_ports.py` |

### Phase 4 — Migration and production rollout

| # | Task | Why it was required | What was implemented | Main evidence |
|---:|---|---|---|---|
| 37 | Controlled data migration | Existing records may rely on the shared legacy timestamp and had no explicit data schema version or safe reprocess workflow. | Added schema version `valdDataSchemaVersion: 2`, canonical checksummed `0600` backups, global duplicate audit, optimistic compatible migration, explicit-ID watermark staging, immutable batch manifest, and before/after reconciliation. The old reset script is now only a safe compatibility entry point. | `migrate_vald_exercise_data.py`, `src/utils/controlled_migration.py`, `docs/16-controlled-data-migration.md`, `tests/test_controlled_migration.py` |
| 38 | Production rollout | The exercise deploy removed the current container before validating the replacement and could not automatically recover. | Added clean-commit releases, `git archive`, SHA-256 verification, server-owned environment, OCI/release metadata, scheduler lock, loopback candidate readiness, approved canary reconciliation, retained rollback container, promoted readiness gate, automatic rollback, and release-specific retained logs. The VALD deployment now has one authoritative entry point. | `deploy_vald_exercise.sh`, `deployment/deploy_vald_exercise_remote.sh`, `deployment/validate_release_canary.py`, `docs/17-production-rollout.md`, `tests/test_release_canary.py` |

## Important compatibility decisions

### MongoDB collections

- Active patient exercise data remains in `stance-dashboard.vald-exercise-data`.
- The removed legacy helper wrote to `stance-dashboard.matches`; the active
  exercise workflow no longer writes there.
- The profile-management service remains a separate concern and was not merged
  into the exercise-data handler.

### Watermarks

- New synchronization uses independent ForceDeck and ForceFrame watermarks.
- A missing product watermark temporarily falls back to `lastRecordedUtc`, so
  legacy records remain readable before controlled migration.
- New success updates do not write the shared legacy timestamp.
- Watermarks advance only after the related data is persisted.

### Cached response compatibility

- Existing top-level and unrelated fields are preserved.
- Existing Dynamo content is preserved but not refreshed.
- Internal `_sourceTestIds` support replay safety and history reconciliation.
- Public WebSocket responses use schema `2.0` and remove internal metadata where
  required by the response path.

### Migration and deployment

- Migration/index application is never automatic at application startup.
- Historical reprocessing requires an exact reviewed bundle and explicit IDs.
- Production canary execution is restricted in three places to
  `683da5b2e2467bc63941a172`.
- The deploy script requires a clean committed checkout. The feature branch is
  committed; pull-request approval and CI are the next gates before deployment.

## What has not been executed or independently proven

State these points clearly in the meeting:

| Item | Current status | Required next action |
|---|---|---|
| Credential rotation | Source credentials/fallbacks were removed; external credential rotation was not performed here. | Confirm whether any historical credential was real, then rotate it in MongoDB/secret management. |
| Live MongoDB duplicate/shape audit | Audit tooling exists; it has not been run against production in this implementation stage. | Run read-only audit and review results before creating indexes or migration bundles. |
| Production indexes | Definitions and idempotent apply tooling exist; indexes were not created. | Run `manage_vald_indexes.py` in audit mode, review impact, then explicitly apply. |
| Production document sizes | Read-only audit and write guard exist; production distribution is unknown. | Run `audit_vald_document_sizes.py` and agree retention/normalization policy if documents approach thresholds. |
| Live VALD pagination/rate limits | Behaviour is fixture-tested; the live tenant contract and quotas were not independently verified. | Confirm cursor semantics, endpoint versions, and quotas with the previous developer/VALD documentation. |
| Frontend contract compatibility | The response is versioned and unit-tested; the real frontend was not run in this workspace. | Validate schema `2.0`, boolean fields, error statuses, and cached output against staging frontend. |
| Controlled migration | Workflow is implemented and offline-tested; no patient record was migrated or reset. | Prepare the approved canary bundle, review backup, apply schema, stage reprocess, and reconcile. |
| Production rollout | Candidate/canary/rollback workflow is implemented and offline-tested; it has not run on EC2. | Configure server-owned environment and infrastructure file, then perform an approved staged rollout. |
| Scheduler authority | Both cron and Lambda integration are supported through one runner; which trigger is enabled in production is unknown. | Confirm one authoritative production trigger and disable the duplicate. |
| Load/performance results | Concurrency is bounded and behaviour is unit-tested; no representative production load test was run. | Agree workload/rate limits and run staging load tests before increasing concurrency. |

## Questions to ask the previous developer

1. Is `stance-dashboard.vald-exercise-data` definitively the only collection the
   exercise WebSocket should mutate?
2. Can a patient intentionally have more than one active `associatedValdId`, or
   does that always indicate bad mapping data?
3. What exact business meaning was intended for `lastRecordedUtc`: recorded time,
   modified time, last fetch time, or last processing time?
4. Do live ForceDeck and ForceFrame endpoints paginate exclusively through
   `ModifiedFromUtc`, and can multiple records share the same cursor timestamp?
5. What are the confirmed tenant request limits for token, tests, trials, and
   metrics endpoints?
6. Does the frontend depend on any string boolean, legacy error message, internal
   field, or exact ordering not represented in the test fixtures?
7. Are legacy cached sessions known to exist without `sessionDate`, `sessionDates`,
   or source IDs? If yes, what should be used to prove no historical loss?
8. Are duplicate `stanceId` records known in production, and how were they handled
   operationally before this work?
9. Is the current production VALD container deployed from the standalone
   `vald-exercise-extract` directory or the versioned
   `oneview-app/vald-exercise-releases` path, and which path should become
   authoritative after review?
10. Is cron or EventBridge/Lambda currently authoritative for scheduled batches?
11. Is `MONGO_READ_URI` a replica connection, and what replication delay is
    acceptable when returning data immediately after a write?
12. Are TLS paths, remote base directory, container name, and UID/GID permissions
    compatible with the production host?
13. Is the approved canary ID still `683da5b2e2467bc63941a172`, and may the canary
    legitimately update that record during rollout?
14. What log-retention duration and backup retention have been approved?

## Likely challenges and concise answers

### “Why was this such a broad change?”

The issues were connected. Fixing performance before persistence ordering,
watermarks, deduplication, and concurrency could make data loss happen faster. The
implementation therefore established regression coverage and correctness first,
then optimized fetch/database behaviour, then hardened operations and rollout.

### “Why are CSV and PNG files being added?”

The production/server working tree contained these parser mapping and graph-design
reference assets, but the target release did not track them. The parser requires
the three CSVs to assign graph types and deterministic defaults. Versioning their
schemas and hashes prevents different machines or images from silently producing
different responses. The PNGs document the graph catalogue and are integrity-
checked reference assets; they are not served to clients or used as patient data.

### “Did we manually change thousands of CSV rows?”

No. The CSV content was imported from the production/server code snapshot. The PR
shows thousands of added lines because Git is seeing previously untracked data
files for the first time. The implementation change around them is the manifest,
strict validation, and regression coverage.

### “Why not continue using one `lastRecordedUtc`?”

ForceDeck and ForceFrame advance independently. One shared maximum can cause the
slower product to skip valid data. Independent product watermarks remove that
cross-product coupling while legacy fallback keeps old documents readable.

### “Why was Dynamo removed?”

Dynamo was explicitly declared future scope. It remains implemented and cached
Dynamo data is preserved, but it cannot trigger current fetch decisions or move
ForceDeck/ForceFrame watermarks.

### “Why reject multiple VALD associations instead of choosing the first?”

Array ordering is not a safe identity decision. Choosing the first association
could fetch and store another athlete’s data. Ambiguity now fails visibly so the
mapping can be corrected.

### “Why store `_sourceTestIds`?”

Timestamp alone is insufficient for replay and equal-timestamp records. Stable
source IDs make merges idempotent, preserve history, and provide migration/canary
reconciliation evidence. They are internal metadata, not a frontend business field.

### “Why not create indexes automatically?”

A unique index can fail or cause operational impact when duplicates exist. The
tool audits first and requires explicit apply so duplicate resolution and production
timing remain controlled.

### “Why does deployment run a real canary?”

Readiness proves dependencies are reachable, not that one full VALD-to-parser-to-
Mongo flow preserves history. The canary covers that path, but is restricted to the
single approved ID and blocks promotion if prior trace tokens disappear.

### “Can we deploy this worktree immediately?”

Not directly to production. The branch is clean and committed, but it must first
pass pull-request review and both CI jobs. Then configure the server-owned
environment and perform the documented candidate, canary, readiness, and rollback
gates. Index or migration application is a separate explicit operation.

## Recommended meeting flow

1. Confirm current production architecture, entry points, collection ownership,
   scheduler, and deployment command.
2. Review Phase 1 correctness changes in `vald_exercise_db.py`,
   `exercise_history.py`, `data_updater.py`, and the response model.
3. Review Phase 2 VALD request behaviour, rate assumptions, Mongo pooling, leases,
   and batch manifest/retry design.
4. Review Phase 3 security boundaries, readiness, CI, container runtime, scheduler,
   and removed legacy paths.
5. Review Phase 4 migration and rollout runbooks without executing them.
6. Record answers to the open compatibility questions.
7. Agree on commit/PR structure, staging owner, backup owner, canary approval, and
   rollback owner.

## Pre-push checklist

- Confirm the intended target branch and create a reviewable feature branch/PR.
- Review every deletion, especially the legacy Mongo helper, file-based VALD
  runner, obsolete launcher, and timestamp implementation note.
- Ensure `.env`, `deployment/infrastructure.env`, migration bundles, logs, patient
  exports, archives, and keys are not staged.
- Run `bash deployment/validate_release.sh` from a clean checkout.
- Confirm CI passes all quality and container jobs.
- Do not apply indexes, migrations, timestamp resets, or production deployment as
  part of the code push.

## Approval sequence after code review

```text
Code review and CI
        |
        v
Staging configuration/readiness
        |
        v
Read-only Mongo audits and backup approval
        |
        v
Approved canary migration/reprocess/reconciliation
        |
        v
Immutable candidate deployment and release canary
        |
        v
Health-gated promotion or automatic rollback
        |
        v
Post-release monitoring and evidence review
```
