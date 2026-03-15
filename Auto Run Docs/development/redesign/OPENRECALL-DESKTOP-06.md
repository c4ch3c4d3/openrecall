# OPENRECALL-DESKTOP-06 - Privacy Controls, Deletion, Export, and Hardening

## Scope

This document defines privacy/data-management UX, deletion semantics, export behavior, migration/backfill tasks, test coverage, packaging, release checks, and end-to-end acceptance criteria for the desktop product.

## Required Outcomes

- [x] Define the privacy/data-management area in Settings
- [x] Specify the delete-everything flow and confirmation requirements
- [x] Specify selective deletion by time range
- [x] Define deletion semantics across all stored artifacts
- [x] Define operations-log export behavior and formats
- [x] Specify confirmation, progress, and completion UX for destructive actions and exports
- [x] Define migration and backfill tasks for existing users
- [x] Define automated test coverage expectations
- [x] Define packaging and release tasks
- [x] Define the end-to-end acceptance pass before release

## Privacy and Data Management UX

### Settings Area
- Entry point: `Privacy/Data Management` remains a dedicated top-level route in the desktop navigation, with deep links from Search, Timeline, AI Setup, and Settings when a user starts an export, reviews migration issues, or clears provider credentials. The route opens on an overview panel rather than a destructive dialog.
- Visibility and discoverability: The overview panel shows storage usage, current retention mode, most recent delete job, most recent export job, unresolved migration items, and provider credential status as separate cards before any destructive controls appear. High-risk actions are grouped below the summary cards and labeled with explicit `Deletes data`, `Exports data`, or `Clears credentials` badges.
- Separation from routine settings: Routine settings may reveal the storage folder or show usage totals, but they cannot host inline delete buttons, export destinations, or credential clearing. Any contextual action elsewhere in the app must navigate into this area with the scope prefilled and clearly restated before execution.

### Delete Everything Flow
- Confirmation copy: `Delete all captured history from this device? This removes screenshots, OCR text, embeddings, summaries, cached provider outputs, and search index entries managed by OpenRecall. App preferences, diagnostics logs, and provider credentials are not removed by this action.`
- Required typed confirmation or multi-step confirmation: Use a two-step destructive flow. Step 1 shows the exact scope with estimated capture count, storage size, oldest and newest capture timestamps, and an explicit note that the recorder will pause while the deletion snapshot is created. Step 2 requires typing `DELETE ALL HISTORY` and checking `I understand this cannot be undone`.
- Completion state: The action creates a tracked background delete job. On success, show deleted capture count, reclaimed bytes, start and finish times, residual review-needed count if any, and actions to `Resume Recorder`, `Open Storage Folder`, or `View Deletion Report`. Search and Timeline should switch to empty-state messaging immediately after the job reaches committed success.
- Failure state: Partial failures remain visible as `Deletion review needed` items with retry and reveal-path actions. The app must never claim success if screenshots, side-table rows, or index state remain inconsistent. If the database portion succeeds but file cleanup is incomplete, the result is `Completed with manual cleanup required`, not `Success`.

### Selective Deletion Flow
- Time-range selection: Selective deletion is range-based for the first desktop release. Support presets `Today`, `Yesterday`, `Last 7 days`, and a custom local-time start and end. Range interpretation is inclusive on the start bound and exclusive on the end bound. Search or Timeline may deep-link into this flow with the visible date range prefilled, but the resolved deletion scope is always explicit timestamps, not ranked-result ids.
- Preview behavior: Before confirmation, run a dry-scope estimate and show the affected capture count, estimated bytes, first and last affected timestamps, and a day-by-day summary. The preview may show a small sample of affected captures, but it must avoid exposing full OCR text unless the user intentionally expands an item.
- Completion state: Successful range deletion shows the exact time range removed, deleted counts by artifact type, reclaimed bytes, and whether search compaction or reindex cleanup is still finishing in the background. The active Timeline view should refresh to the nearest surviving capture outside the deleted range.
- Failure state: If any chunk fails, preserve the scoped job record, report the exact chunk boundary or capture id range that failed, and allow retry from that point. Missing screenshot files inside the selected range are treated as handled inconsistencies rather than fatal blockers as long as their metadata and side-table rows are removed.

## Deletion Semantics

| Artifact | Delete Everything | Selective Deletion | Notes |
|----------|-------------------|--------------------|-------|
| Screenshots | Hard-delete all managed screenshot files, generated thumbnails, and temporary image derivatives after the job snapshots the target set. | Delete screenshot files for captures whose `captured_at` falls inside the confirmed range. Missing files are recorded and skipped without aborting the whole job. | Only files under the managed media root may be removed. Any path outside that root is blocked and flagged for review. |
| OCR text | Remove raw text, normalized text, excerpts, and language or token metadata for all captures. Remove matching FTS rows in the same job chunk. | Remove text rows and FTS documents for the selected captures only. | Deletion reports may include counts and capture ids, but they must never echo deleted OCR content. |
| Embeddings | Remove vectors for all embedding profiles, active and inactive, plus any auxiliary vector-index state derived from them. | Remove vectors for the scoped captures and decrement profile coverage counters. | Vector indexes are rebuildable implementation details and must never outlive their source records. |
| Summaries | Remove all summary text, tags, entity hints, importance scores, and summary provenance rows. | Remove all derivative rows tied to the selected captures. | If summary text is mirrored into lexical indices, those index documents are removed with the same capture chunk. |
| Search indices | Drop or recreate the lexical and auxiliary search index state as empty after delete-all, then clear cached search metadata. | Delete matching lexical documents immediately and schedule compaction or a full rebuild when the removed share crosses the configured threshold. | Search may show `Maintenance in progress` until compaction finishes, but deleted captures must disappear from results as soon as the delete job commits. |
| Cached provider outputs | Remove cached provider responses, temporary upload payloads, export staging files, and capability artifacts tied to deleted captures. | Remove only cache entries associated with captures in the selected time range. | Provider credentials, diagnostics bundles, and routine settings are outside capture deletion and require separate user actions. |

### Deletion Guarantees

- Delete jobs may use internal soft-delete markers while work is in progress, but the user-facing completion state must represent hard removal of the selected history.
- `Delete Everything` applies to captured history and capture-derived artifacts only. It does not clear routine settings, local diagnostics logs, or OS-backed provider credentials.
- Delete jobs are chunked, resumable, and idempotent. Re-running the same confirmed job after a crash must not recreate or double-count deleted state.
- Export jobs targeting overlapping captures are blocked once a delete job enters the committed phase, and new capture intake pauses only for the short snapshot window required to freeze scope.

## Operations Log Export

### Export Goals
- Timeline-style artifact from screenshots only: The export is a chronological reconstruction derived only from recorded captures and their local metadata. It never joins browser history, shell history, clipboard data, or remote provider logs. Screenshots remain the canonical evidence source, while OCR text and summaries are optional derived annotations.
- Timestamp handling: Every exported record carries `captured_at_local`, `captured_at_utc`, and the original UTC offset. Human-readable Markdown shows local time with timezone label; JSON preserves ISO 8601 UTC and local-offset fields so imports and audits stay stable across machines.
- CLI-command inference: When app metadata, window title, and OCR strongly indicate a terminal session, the exporter may emit an `inferred_command` field with a confidence label. Low-confidence detections fall back to literal OCR excerpts. The exporter must never invent hidden arguments, assume command success, or present inferred commands as authoritative shell history.
- Non-CLI activity summaries: Non-terminal sequences are grouped into chronological activity blocks using app or window metadata, OCR excerpts, and derived summaries when available. If summaries are unavailable or stale, the exporter falls back to literal metadata plus OCR snippets instead of withholding the records.

### Export Formats

| Format | Purpose | Included Fields | Notes |
|--------|---------|-----------------|-------|
| Markdown | Human-readable activity report for review, sharing, or incident notes. | Export metadata header, selected scope, day and session headings, local timestamps, app or window labels, OCR excerpts or summaries, inference labels, and relative screenshot references when image copy is enabled. | Default format. Redact obvious secret-like tokens by default and annotate when redaction occurred. |
| JSON | Structured archive for automation, audit tooling, or future reimport helpers. | Export manifest metadata, capture ids, timestamps, relative asset paths, app and window metadata, processing-state markers, OCR fields allowed by export options, summary fields, inference metadata, and checksums. | Best for tooling. Raw OCR text requires an explicit advanced opt-in; normalized excerpts remain the default textual payload. |
| Hybrid JSON + Markdown | Combined human-readable and machine-readable export package. | Markdown report, JSON manifest, checksum file, and copied screenshot assets under a stable folder structure. | Recommended for investigations or handoff workflows. Package as a folder by default, with optional zip creation as a follow-up background step for large exports. |

### Export UX
- Confirmation: Show scope, estimated capture count, estimated bytes, chosen format, destination path, whether screenshots are copied, whether raw OCR is included, and whether privacy redaction is enabled. Exporting outside the managed data root requires a second confirmation through Electron main because it crosses the trusted storage boundary.
- Progress: Exports always run as tracked background jobs with visible phases such as `Collecting captures`, `Rendering activity blocks`, `Copying screenshots`, `Writing manifest`, and `Finalizing checksums`. Users may keep browsing while export runs, but overlapping delete jobs are blocked until export finishes or is canceled safely before the copy phase.
- Completion: Successful export shows output path, file count, byte size, checksum manifest location, redaction summary, and actions to `Open Export Folder`, `Copy Manifest Path`, or `Start New Export`. Completion UI must distinguish between `Export created` and `Export created with skipped captures` when corrupt or missing screenshot files were omitted.
- Large-dataset handling: Stream export work in day-sized chunks. When the estimated result exceeds the configured disk or file-count soft threshold, force the hybrid folder layout, warn about size before confirmation, and keep optional zip compression as a separate post-processing step so the primary export can complete without a single large-memory operation.

## Migration and Backfill

- Existing-user migration tasks: Detect the legacy `recall.db` and timestamp-named screenshot files in the current OpenRecall storage root or a user-confirmed custom storage path. Before writing any new schema data, create a timestamped backup of the legacy database, import canonical capture rows, verify screenshot file presence, move or map assets into the managed media root, and place unresolved assets into a visible migration-review queue rather than silently dropping them.
- Reindex or resummarization requirements: Rebuild lexical search state for all migrated records because the desktop schema and FTS model change from the current Flask app. Reuse embeddings only when stored vector dimensions and profile metadata match the approved active embedding profile; otherwise queue re-embedding after baseline text import. Provider-generated summaries may be imported only as stale derivatives and must be regenerated opportunistically when their version or provider contract no longer matches.
- Upgrade-path checks: Validate clean install, legacy Flask import, upgrade from prior desktop beta schemas, custom-storage migration, interrupted delete-job resume, interrupted export-job cleanup, and provider-setting preservation. Delete and export controls stay disabled until migration reaches the minimum safe state of canonical capture rows plus verified search text availability.
- Rollback considerations: Keep the legacy database and legacy screenshot directory untouched until the new desktop schema passes integrity checks and records a committed migration version. If migration fails, the app enters a read-only remediation state with retry diagnostics; it must not partially switch live writes back to the legacy Flask schema. For upgrades between desktop builds, keep the previous schema backup until the new build has launched successfully at least once.

## Automated Tests

| Area | Coverage Goal | Test Type | Notes |
|------|---------------|-----------|-------|
| Configuration | Verify route separation, retention defaults, path-token handling, and that destructive controls require explicit confirmation state before the API accepts them. | Unit plus renderer-service integration tests | Include assertions that the renderer never receives raw secrets, raw filesystem handles, or unvalidated delete/export scopes. |
| Provider selection | Verify `Disabled` remains the default, remote trust acknowledgements persist correctly, credential-clear flows are isolated from history deletion, and provider state survives upgrades. | Integration tests | Clearing credentials must not delete capture history, and deleting capture history must not clear OS-backed credentials. |
| Search/index behavior | Verify migrated data indexes correctly, deleted captures disappear from search and timeline results, and reindex or compaction state is reported after deletion. | Integration plus migration tests | Cover lexical-only mode, hybrid mode, and missing-embedding degraded mode after selective delete and delete-all operations. |
| Deletion semantics | Verify range-boundary handling, chunk retries, crash recovery, orphan screenshot handling, and delete-all consistency across screenshot files and side tables. | Unit, integration, and crash-recovery tests | Include local-time edge cases such as daylight-saving transitions and missing screenshot files. |
| Export generation | Verify Markdown, JSON, and hybrid packages; timestamp serialization; redaction behavior; checksum output; screenshot copying; and skipped-capture reporting. | Integration plus snapshot tests | Cover terminal inference labeling, non-terminal activity grouping, and exports with AI disabled. |
| Recorder control APIs | Verify recorder pause or resume behavior around delete and export jobs, health-status propagation, and safe rejection of destructive work during migration. | Integration plus end-to-end tests | Tray state, shell status chips, and service job banners must agree on recorder and maintenance state. |

### Test Gate

- No desktop release is ready until delete-all, selective deletion, and export generation coverage runs in automated CI on the primary Windows target.
- At least one crash-recovery test must prove that an interrupted delete or export job resumes or fails safely without corrupting canonical capture rows.
- Manual release sign-off cannot waive missing automated coverage for destructive flows; the most manual testing can do is supplement the soak and packaging checks.

## Packaging and Release

- Desktop packaging tasks: Ship a signed Windows installer as the first supported desktop package, alongside a portable developer-friendly artifact if needed for internal testing. The package must include Electron main, preload, renderer assets, the bundled Python service, baseline OCR dependencies, migration helpers, license files, and uninstall metadata. The uninstaller should preserve user data by default and offer a separately confirmed `Remove local OpenRecall data` option rather than deleting history silently.
- Local service bundling: Bundle the Python runtime and all dependencies required for local-only capture, OCR, migration, delete, and export flows. Optional provider integrations rely on user-supplied local or remote endpoints and must not bundle secrets or remote credentials. The packaged service boot path, schema version, and health probe contract must be pinned to the desktop build that launches it.
- First-run validation: On a clean Windows account, validate storage-root creation, tray initialization, onboarding, local-only recorder start, timeline availability without AI, diagnostics export, and privacy-route access before any provider is configured. The first-run flow must also handle legacy-data detection and incomplete-migration states without crashing or trapping the user in a dead-end screen.
- Upgrade-path validation: Validate upgrades from the current Flask storage layout, from a prior desktop beta with existing settings and credentials, and from a build interrupted during delete, export, or migration work. Upgrade validation must confirm that old backups remain accessible, migrated data stays searchable, and provider state or recorder preferences do not regress silently.

## Hardening Requirements

- Runtime boundaries: Production builds disable devtools by default, load only packaged local assets, enforce a strict renderer content-security policy, and reject any unexpected remote content sources.
- IPC and API validation: Every preload IPC call and loopback API request must use versioned schema validation. Reject malformed payloads, missing session tokens, path traversal attempts, and symlink or junction destinations outside the managed data root or the user-approved export target.
- Filesystem safety: The Python service may write only inside the managed app-data root or a path token explicitly approved through Electron main for export. Delete jobs may remove only managed media-root files, never arbitrary user-selected directories.
- Secret and log hygiene: Provider credentials remain in the OS credential store only. They must not appear in exports, delete reports, logs, backups, or migration manifests. Production logging defaults to operational metadata only, with higher local verbosity available solely through explicit user action.
- Release hygiene: Pin Electron and Python dependency versions, run dependency and license audits before packaging, verify code signing and timestamping on release artifacts, and scan the packaged binaries for unexpected network endpoints or bundled secrets.

## End-to-End Acceptance Pass

- Long-running background capture: Run a Windows soak pass of at least 24 hours covering active use, idle periods, sleep or wake, lock or unlock, tray-only operation, and one supervised service restart. Recorder state, queue depth, storage growth, and service health must remain stable without silent capture stoppage.
- AI setup flexibility: Validate a full local-only path with no provider configured, then enable and disable each supported provider mode without breaking capture, timeline browsing, lexical search, delete flows, or export flows. Failed provider tests must leave the app usable and clearly degraded rather than blocked.
- Hybrid search quality: Verify that migrated and newly captured records are searchable through lexical-only mode and hybrid mode, that deleted captures disappear from both ranked search and timeline navigation immediately after job commit, and that search performance remains within the Phase 04 targets on the benchmark dataset.
- Timeline review: Verify date jumping, day grouping, detail-panel navigation, surrounding-capture browsing, and deep links into delete or export flows from the current time range. The timeline must remain usable when AI-derived summaries are absent or stale.
- Deletion flows: Validate selective deletion across ordinary ranges and daylight-saving boundaries, full-history deletion, manual-cleanup reporting for missing files, recorder pause and resume behavior, and post-delete empty states. Also verify that clearing provider credentials remains a separate flow and does not piggyback on history deletion.
- Operations-log export: Generate Markdown, JSON, and hybrid exports for small and large datasets, confirm timestamps and checksum manifests, confirm optional screenshot copying and redaction behavior, and verify that exports remain readable outside the app. Large exports must finish through streamed chunking rather than exhausting memory.

## Approved Decisions

- `Delete Everything` removes captured history and capture-derived artifacts, not app preferences or provider credentials.
- Destructive actions and exports are always asynchronous tracked jobs with resumable progress and explicit completion summaries.
- Operations-log export is local-only and derived from captures plus their local metadata; OpenRecall does not join external activity sources for this feature.
- Windows is the first packaging target, but delete, export, migration, and hardening contracts must stay portable across future platforms.

## Open Questions

- Whether the first desktop release should keep retention strictly manual, or also expose scheduled auto-deletion once the delete-job recovery path has enough soak coverage.
- Whether the hybrid export package should eventually support built-in password-protected archives, or continue relying on user-managed encrypted storage volumes for secure sharing.

## Acceptance Criteria

- Delete-all and selective-deletion semantics are complete and internally consistent
- Export formats and UX are defined clearly enough for implementation
- Migration, testing, packaging, and acceptance criteria are explicit before release work begins
- Hardening requirements are concrete enough to guide production packaging and release review
