# OPENRECALL-DESKTOP-02 - Architecture and Foundations

## Scope

This document defines the technical foundation for moving OpenRecall from the current Python/Flask UI to an Electron desktop shell with a React/TypeScript renderer and a Python local background service.

## Required Outcomes

- [x] Audit the current Python/Flask app structure, screenshot pipeline, OCR flow, SQLite schema, and configuration flow
- [x] Decide what remains as the long-running local service and what is retired with the Flask UI
- [x] Define the target multi-process architecture and process boundaries
- [x] Document startup order, shutdown behavior, failure modes, and long-running background expectations
- [x] Specify the backend service surface required by the desktop app
- [x] Define the settings model and its domains
- [x] Decide the persistence strategy and migration requirements
- [x] Define background-efficiency rules and resource-management constraints
- [x] Establish security and privacy baselines
- [x] Record the approved implementation sequence before major code changes begin

## Current-State Audit

### Application Structure
- Current UI entry points: `openrecall/app.py` owns the entire user experience today. It defines the Flask app, inline Jinja templates, timeline route (`/`), search route (`/search`), and static screenshot serving (`/static/<filename>`), then starts both the web server and the recorder thread in the same Python process.
- Current recorder entry points: `openrecall/screenshot.py` provides `take_screenshots()` and a recorder loop named `record_screenshots_thread()`. The intended loop captures every 3 seconds, skips work while the user is idle, uses MSSIM to avoid duplicate frames, writes WebP screenshots to disk, runs OCR, computes embeddings, and inserts metadata into SQLite. The file currently contains two conflicting `record_screenshots_thread()` implementations, which means recorder behavior is not cleanly isolated and must be normalized before reuse.
- Current OCR and indexing flow: `openrecall/ocr.py` loads a global Doctr OCR predictor at import time and extracts plain text synchronously from each accepted screenshot. `openrecall/nlp.py` loads a global `SentenceTransformer` model at import time, builds one mean embedding per OCR result, and search computes the query embedding inside the request handler before comparing it against every row already loaded from SQLite.
- Current storage and schema notes: `openrecall/database.py` creates a single `entries` table in SQLite with columns `id`, `app`, `title`, `text`, `timestamp`, and `embedding`, plus an index on `timestamp`. Screenshots are stored as WebP files under a screenshots folder. The current code assumes filenames derived from timestamps, but the recorder code and Flask UI do not consistently agree on filename shape, and the database does not store screenshot paths or monitor identity.
- Current configuration flow: `openrecall/config.py` parses CLI arguments at import time, computes `appdata_folder`, `db_path`, and `screenshots_path` as module globals, and creates folders as a side effect. There is no persisted settings model beyond process startup arguments, no separation between user-editable preferences and runtime state, and no secure secret storage for future provider credentials.

### Retain vs Retire Table

| Area | Keep as Service | Retire with Flask UI | Notes |
|------|-----------------|----------------------|-------|
| UI routing | No | Yes | Replace Flask route rendering with Electron main + React renderer. Browser templates, CDN Bootstrap, and inline HTML are retired. |
| Recorder loop | Yes | No | Keep capture orchestration, active-window inspection, idle detection, and frame-dedup logic in the Python background service. Refactor away from Flask-coupled thread startup. |
| OCR processing | Yes | No | Keep local OCR and embedding generation in Python, but move to a queue-driven service pipeline with explicit worker ownership and health reporting. |
| Search APIs | Yes | Yes | Keep search and timeline domain logic as service capabilities, but retire Flask HTTP handlers and server-rendered search results. |
| Settings handling | Yes | Yes | Keep configuration as a service-owned domain model, retire import-time argparse globals, and move secrets into OS-backed secure storage managed through Electron. |

## Target Architecture

### Process Model

| Process | Responsibilities | Inputs | Outputs | Failure Handling |
|---------|------------------|--------|---------|------------------|
| Electron main | Own native app lifecycle, single-instance lock, tray, window creation, preload boundary, service bootstrap, secure secret mediation, auto-start integration, and packaging hooks. | OS lifecycle events, renderer IPC calls, service health probes, persisted app bootstrap config. | Renderer window state, tray/status notifications, keychain reads and writes, service launch/stop commands, recovery prompts. | If the renderer crashes, recreate the window and keep the service alive. If the service fails, show degraded-state UI, attempt bounded restarts, and preserve logs needed for local diagnostics. |
| Renderer (React/TS) | Render the desktop UI, onboarding, settings, recorder controls, timeline, search, and explanatory status. Hold only view state and non-secret cached preferences. | Typed preload IPC, loopback service API responses, status stream updates, local navigation state. | User commands, settings edits, search queries, delete/export requests, visible health/error states. | If the renderer fails, Electron main rehydrates it from persisted window state without restarting capture. Renderer must tolerate service-unavailable and migration-in-progress states. |
| Python background service | Own capture scheduling, screenshot persistence, OCR, embeddings, search/timeline query execution, migration, retention jobs, and export/delete back-end work. | Service config, capture control commands, local files, SQLite data, optional provider requests. | Domain API responses, recorder status events, migration progress, export artifacts, structured diagnostics. | Run as a supervised child process. Persist in-flight job markers so startup can reconcile incomplete OCR/index jobs after crash or reboot. |

### Startup and Shutdown Sequence

1. Application launch: Electron main starts first, acquires the single-instance lock, loads bootstrap settings from disk, and creates the tray placeholder before any renderer window is shown.
2. Service bootstrap: Electron main launches the Python service as a child process, negotiates an ephemeral loopback port and session token, and waits for a ready health check with a bounded timeout. If the service is upgrading data, main exposes a migration state instead of normal navigation.
3. Renderer hydration: Once the preload bridge has the service connection metadata, the React renderer loads shell chrome, pulls the initial recorder status and settings snapshot, and then hydrates the current route.
4. Background/Tray transition: Closing the main window minimizes to tray by default while leaving the Python service alive. Explicit Quit from tray or menu stops both renderer and service.
5. Graceful shutdown: Electron main asks the Python service to stop capture intake, drain any active OCR/index transaction, flush SQLite writes, and acknowledge safe shutdown before the child process is terminated.
6. Recovery after crash or restart: On unexpected service exit, main attempts a limited restart cycle and surfaces a clear degraded-state banner. On app restart, the service reconciles incomplete queue items, validates screenshot files referenced by unfinished jobs, and resumes normal capture only after storage integrity checks pass.

### IPC and API Boundaries

| Boundary | Direction | Transport | Contract | Notes |
|----------|-----------|-----------|----------|-------|
| Main -> Renderer | Push | Electron preload IPC (`contextBridge`) | Window lifecycle events, tray actions, keychain availability, service connection metadata, migration progress, and privileged-operation results. | Renderer never receives raw Node.js access. All contracts must be typed and versioned with a small preload API surface. |
| Renderer -> Main | Invoke | Electron preload IPC | Open/minimize/quit actions, reveal storage folder, import/export file chooser, launch-at-login toggles, and secret-write requests. | Any operation needing filesystem dialogs, OS integration, or secret storage stays on this boundary. |
| Main -> Python service | Request/command | Loopback HTTP/JSON plus supervised process stdin/stdout for boot logs | Health checks, shutdown command, migration command, diagnostics level changes, and startup configuration handoff. | Main is the process supervisor and only actor allowed to start or stop the service binary. |
| Renderer -> Python service | Request/subscribe | Loopback HTTP/JSON for CRUD and Server-Sent Events for status stream | Recorder controls, settings reads/writes, timeline/search queries, delete/export jobs, provider tests, and health/status subscription. | Main passes an ephemeral token to the renderer. Service listens only on `127.0.0.1` and rejects requests without the token. |

## Backend Service Surface

| Capability | Endpoint or IPC Contract | Request | Response | Notes |
|------------|---------------------------|---------|----------|-------|
| Recorder status | `GET /v1/recorder/status` | None | `{ running, mode, capture_interval_seconds, last_capture_at, backlog_depth, active_provider_mode }` | Source of truth for tray status and shell indicators. |
| Start recorder | `POST /v1/recorder/start` | Optional `{ reason }` | Updated recorder status snapshot | Idempotent. Reject if migration or destructive maintenance job is active. |
| Stop recorder | `POST /v1/recorder/stop` | Optional `{ reason }` | Updated recorder status snapshot | Supports manual pause from UI and shutdown drain flow. |
| Capture interval settings | `PATCH /v1/settings/recorder` | `{ capture_interval_seconds, primary_monitor_only, pause_when_idle, launch_on_login }` | Persisted recorder settings snapshot | Electron main applies `launch_on_login`; service owns capture settings. |
| AI provider configuration | `GET/PATCH /v1/settings/providers`, `POST /v1/providers/test` | Provider mode, endpoint, model names, capability flags, secret references | Sanitized provider config, capability report, health result | Secrets are referenced by key ID only; raw keys never leave Electron main or OS keychain. |
| Timeline queries | `GET /v1/timeline` | Date range, pagination cursor, app/window filters, include thumbnails flag | Timeline items with screenshot refs, timestamps, app/window metadata, OCR excerpt, and processing state | Timeline remains useful with AI disabled. |
| Search queries | `POST /v1/search` | `{ query, mode, filters, limit, cursor }` | Ranked results with lexical score, semantic score, match reasons, screenshot refs, and snippets | Explainability fields are mandatory even before reranking is added in Phase 04. |
| Delete flows | `POST /v1/delete-jobs`, `GET /v1/delete-jobs/:id` | Scope selector, confirmation token | Job id, progress, outcome summary | Detailed semantics are defined in Phase 05; this phase locks the service boundary and async job shape. |
| Export operations | `POST /v1/export-jobs`, `GET /v1/export-jobs/:id` | Export scope, format, target path token | Job id, progress, artifact manifest | Renderer requests a file target through Electron main, then passes the approved token/path handle to the service. |
| Health/status | `GET /v1/health`, `GET /v1/status/stream` | Optional component filter | `{ service, db, capture, ocr, embeddings, provider, migration }` or streamed events | Health is user-facing. Avoid opaque "running" states that hide degraded OCR or indexing. |

## Settings Model

| Domain | Settings Included | Storage Location | Editable From | Notes |
|--------|-------------------|------------------|---------------|-------|
| General app settings | Window state, tray/minimize behavior, launch-on-login, diagnostics level, theme, onboarding completion, current schema version | `settings.json` in app data for non-secret values; OS-managed login item state via Electron main | Renderer via preload IPC | Keep shell UX preferences separate from recorder/search domain settings so the service can run headless when needed. |
| Recorder settings | Capture interval, monitor scope, idle detection toggle, pause schedule, retention defaults, auto-start recorder on launch | `settings.json`, mirrored into service runtime snapshot | Renderer via service API | Service validates ranges and can clamp unsupported monitor settings per platform. |
| Search/index settings | OCR language/profile, indexing enablement, queue limits, FTS/embedding rebuild state, result-size defaults | `settings.json` plus SQLite metadata tables for operational counters | Renderer via service API | Separate mutable preferences from derived index state to keep migrations deterministic. |
| AI provider settings | Provider mode, endpoint URL, model identifiers, capability toggles, test status, local/remote trust flags | Non-secret metadata in `settings.json`; secrets in OS keychain/credential manager | Renderer via preload IPC and service API | Provider mode must support "Disabled" as a first-class default. |

## Persistence and Migration

### Persistence Strategy
- Metadata: Use a versioned SQLite database as the durable source of truth. Phase 01 approves expanding beyond the current `entries` table to normalized capture/session/settings metadata tables, while preserving a compatibility import path from the legacy schema.
- Screenshot image paths: Store screenshots on disk under an app-data managed media root using stable relative paths (`captures/YYYY/MM/DD/<capture_id>.webp`). Persist the relative path, monitor id, dimensions, and hash in SQLite so the UI never has to infer filenames from timestamps.
- OCR text: Persist extracted text and normalized excerpts in SQLite, and maintain an FTS5 virtual table for lexical lookup. OCR job state belongs in durable queue tables so unfinished work can resume after crash.
- Embeddings: Store embeddings in a dedicated SQLite table keyed by capture/item id, versioned by model/provider so re-embedding can occur incrementally when provider settings change.
- Search indices: Use SQLite FTS5 for lexical retrieval in the first desktop release. Any approximate vector index introduced later must be rebuildable from SQLite metadata and embeddings without mutating screenshot assets.
- Provider settings: Store non-secret provider metadata in `settings.json` and store secrets only in the platform credential store mediated by Electron main.

### Migration Requirements
- Existing SQLite compatibility: The desktop service must detect the legacy `recall.db` schema and import from it without requiring the Flask app to still run. Legacy `entries` rows remain readable as source material, but the desktop release does not continue writing to the old schema.
- Backfill or reindex requirements: First desktop launch creates a new schema version and runs an import/backfill job that copies legacy rows, validates screenshot file existence, infers missing screenshot paths only when they match the old timestamp-based convention, and marks unresolved assets for user review. Embeddings may be reused only when their stored vector length matches the active embedding profile; otherwise queue a re-embed job.
- Rollback strategy: Before any migration writes, create a timestamped backup of the original SQLite file and leave screenshot assets untouched. If migration fails, keep the legacy database intact, mark the desktop service as not ready, and offer retry diagnostics rather than partial write-through behavior.
- Mixed-version support during transition: Mixed live writes between Flask and the desktop service are not supported. During rollout, the legacy app is treated as a read-only predecessor and the new desktop service is the only writer once migration succeeds.

## Background Efficiency Plan

- Memory limits: Target steady-state memory budgets of less than 300 MB for Electron main plus renderer and less than 1.5 GB for the Python service with OCR and embedding models loaded. If service memory crosses the soft cap, pause new OCR/embedding work, drain the queue, and surface a degraded-state warning.
- Screenshot retention policy: Only accepted, non-duplicate captures are written to disk. Rejected duplicate frames stay in memory only. Retention remains user-controlled and local-first; no automatic cloud offload or hidden secondary copy is allowed.
- Indexing cadence and batching: Capture intake remains lightweight and enqueues OCR/index jobs instead of doing all work inline with screenshot acceptance. SQLite commits should batch small groups of jobs where safe, but recorder status updates must remain near real time.
- Idle behavior: Pause capture when the user is idle, the session is locked, or the machine is sleeping. Backlog processing may continue briefly after idle if it is CPU-safe, but model-heavy work should yield when the device is on battery saver or repeated thermal pressure is detected.
- Retry behavior: Use bounded exponential backoff for local service failures, provider tests, and transient SQLite lock contention. Do not spin forever on corrupted captures or unreadable files; move them to a failed-job state with user-visible diagnostics.
- Safe startup and shutdown semantics: Startup reconciles queue state before resuming normal capture. Shutdown stops new intake first, then lets active OCR/index steps finish or checkpoint safely. No screenshot file should exist without a corresponding metadata row for longer than one recovery cycle.

## Security and Privacy Baselines

- Local data protection: All capture content, OCR text, embeddings, and metadata remain local by default. Store files only under the user-selected app data root, honor OS filesystem permissions, and make the storage location visible from settings. Future encryption-at-rest support must layer on top of this file layout without changing logical ownership.
- API key handling: Provider credentials are never written to SQLite, plaintext JSON, logs, or renderer state snapshots. Electron main reads and writes secrets through the OS credential manager and hands the Python service only short-lived access tokens or resolved credentials when a provider action actually needs them.
- Local model endpoint trust boundaries: Treat localhost endpoints as opt-in integrations, not implicitly trusted dependencies. Any non-localhost endpoint is labeled remote, requires explicit user acknowledgement, and must be allowed to fail without breaking capture, timeline, or lexical search.
- Confirmation requirements for destructive actions: Stop recorder, clear provider credentials, delete captures, delete all history, and export outside the app-data root all require explicit confirmation with scope shown in human-readable terms. Bulk destructive operations must run as tracked jobs, not one-click fire-and-forget actions.
- Logging and diagnostic limits: Logs must default to operational metadata only: component status, queue counts, error categories, and anonymized file ids. Never log OCR text, screenshot pixels, embeddings, or secrets. Diagnostic bundles remain local and require explicit user export.

## Approved Implementation Sequence

1. Architecture foundation: Stand up the Electron main process, preload boundary, Python service host, versioned settings file, and v2 SQLite schema/migration scaffolding before moving UI behavior.
2. Desktop shell and renderer: Build the React/TypeScript shell, recorder status surfaces, settings information architecture, tray behavior, and lifecycle-aware onboarding against stubbed or minimal service contracts.
3. Provider abstraction: Add optional provider modes, secret handling, capability tests, and service-side model adapters without making AI setup mandatory for capture or browsing.
4. Retrieval and timeline rework: Replace in-memory search ranking with indexed lexical plus semantic retrieval, explainable ranking output, and timeline pagination against the new persistence model.
5. Privacy/export/hardening: Finalize deletion semantics, export jobs, packaging, diagnostics, release validation, and end-to-end long-running stability gates.

## Open Questions

- Windows should be the first packaging target because the existing product framing compares directly against Windows Recall, but macOS and Linux service contracts must not be blocked on Windows-specific shell assumptions.
- The first desktop release should ship SQLite FTS5 plus stored embeddings, but the exact vector indexing library can stay open until Phase 04 as long as it is rebuildable from the canonical SQLite records.
- Migration UX still needs a product decision on whether unresolved legacy screenshot-path mismatches block completion or allow partial import with a remediation list.

## Acceptance Criteria

- The target architecture is fully described
- Every major service/API boundary has an owner and contract outline
- Migration requirements are explicit enough to guide implementation sequencing
- Privacy, security, and background-operation rules are defined before code changes begin
