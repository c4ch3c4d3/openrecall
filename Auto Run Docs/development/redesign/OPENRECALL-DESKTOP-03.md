# OPENRECALL-DESKTOP-03 - Desktop Shell and Core UX Framework

## Scope

This document specifies the desktop shell, application frame, shared UI primitives, recorder controls, settings IA, and desktop lifecycle behaviors for the new OpenRecall experience.

## Required Outcomes

- [x] Define the Electron shell structure and secure preload boundary
- [x] Define local service bootstrapping, packaging path, and development workflow
- [x] Design the primary application frame and navigation
- [x] Specify the shared React/TypeScript UI shell and design-system primitives
- [x] Document recorder controls, status states, and capture-frequency controls
- [x] Cover first-run and returning-user flows
- [x] Define settings IA, tray behavior, startup behavior, and service-health visibility
- [x] Decide whether local-only telemetry or diagnostics are included

## Desktop Shell Architecture

### Shell Components

| Component | Responsibilities | Security Notes | Packaging Notes |
|-----------|------------------|----------------|-----------------|
| Electron main process | Own the single-instance app lock, tray, native window lifecycle, launch-on-login integration, secret storage mediation, file pickers, service supervision, and recovery prompts. | Runs with `contextIsolation` enabled, `sandbox` enabled where supported, `nodeIntegration` disabled, and a strict allowlist of renderer-callable actions. It is the only process allowed to access OS keychain APIs or arbitrary filesystem dialogs. | Shipped as the desktop host binary. The packaged app includes the preload script and the Python service bundle under the app resources directory, with platform-specific installers added later in Phase 05. |
| Preload bridge | Expose a narrow typed API for window actions, tray callbacks, diagnostics export, launch-on-login toggles, storage-folder reveal, and secret write or clear requests. | No raw Node.js objects cross the boundary. Every method validates payload shape and origin, then maps to explicit IPC channels with versioned contracts. | Built with the renderer but emitted as a separate preload artifact. The preload script must remain stable even when the renderer hot reloads in development. |
| Renderer app | Render the search, timeline, recorder, AI setup, settings, and privacy surfaces; subscribe to service health and recorder status; and keep only view state plus non-secret cached preferences. | Treat the preload API and loopback service as the only backends. Never store provider secrets, raw filesystem handles, or privileged Electron objects in renderer state or persisted browser storage. | Packaged as static assets loaded from the Electron app. In development it can run from a local dev server, but the production bundle is file-based and hash-versioned. |
| Local service bootstrap | Launch the Python background service, pass the resolved app-data root plus session token, wait for health readiness, restart on bounded failure, and stop intake during app quit. | The service listens on `127.0.0.1` only and rejects unauthenticated requests. Bootstrap metadata stays in memory only and is never written to logs with secrets attached. | In development, start from the local Python environment and project sources. In packaged builds, start the bundled Python executable or frozen service binary from `resources/service`. |

### Development and Build Workflow
- Local development loop: Run Electron main plus preload locally, point the renderer at a local React development server, and launch the Python service from the checked-out repository so UI and service work can iterate independently. The main process waits for both the renderer dev server and service health endpoint before showing the primary window.
- Packaged build path: Produce a desktop package that embeds the compiled renderer assets, the preload bundle, and the Python service bundle. Phase 02 approves Windows as the first installer target, but the shell contracts must remain portable so macOS and Linux can adopt the same app structure later.
- Local service bundling: Bundle the Python service with pinned dependencies and model bootstrap hooks as a self-contained executable or managed runtime folder under Electron resources. The renderer must never shell out directly to Python; only Electron main can spawn or restart the service.
- Environment handling: User-editable settings come from the versioned settings file plus OS secret storage, not ambient environment variables. Environment variables are allowed only for local development overrides such as dev-server URL, verbose logging, or alternate service binary paths, and the production app ignores them unless launched in an explicit developer mode.

## Application Frame

### Navigation Model

| Area | Purpose | Primary User Actions | Notes |
|------|---------|----------------------|-------|
| Search | Default retrieval workspace for finding prior activity through lexical plus semantic matching, filters, and match explanations. | Enter a query, refine by app or date, inspect why a result matched, open the source capture in context, and jump to the surrounding timeline window. | This is the default landing route after onboarding because it best demonstrates value without requiring the user to understand the timeline model first. |
| Timeline | Visual browse mode for reviewing captures by time with coarse filters and day or session grouping. | Scroll or page through captures, scrub by date, inspect OCR excerpt, and open a detail panel for adjacent captures. | Timeline remains useful even when AI is disabled or indexing is still catching up. |
| Recorder | Operational control center for capture status, interval controls, backlog state, and service health. | Start or pause capture, change interval, see last capture time, inspect warnings, and open the latest diagnostics view. | Recorder status is also visible globally in the shell header and tray, but this route is the detailed control surface. |
| AI Setup | Optional configuration workspace for local or remote model providers and capability testing. | Leave AI disabled, connect a local provider, add a remote endpoint, test capabilities, and review fallback behavior. | The nav label should read `AI Setup` until configuration is complete, then change to `AI Providers` in the UI chrome. |
| Settings | Home for routine app preferences, startup behavior, theme, recorder defaults, and non-destructive operational controls. | Adjust launch behavior, choose storage visibility actions, manage capture defaults, and opt into richer local diagnostics. | Use sectioned settings pages rather than a single long form so the app remains navigable on smaller screens. |
| Privacy/Data Management | Explicit place for deletion, export, retention, migration review, and credential-clearing flows. | Delete a range, export selected history, clear provider credentials, inspect unresolved migration items, and review storage usage. | Separate this area from routine settings so destructive actions are never mixed into everyday preference editing. |

### Shared UI Primitives

| Primitive | Usage | Required States | Notes |
|-----------|-------|-----------------|-------|
| Forms | Settings pages, provider setup, export options, and recorder interval editing. | Default, focused, dirty, saving, saved, validation error, disabled, and degraded-service read-only. | Prefer explicit helper text for privacy or network implications. Secret fields must support test, replace, and clear flows without echoing stored values back to the renderer. |
| Tables | Timeline day summaries, diagnostics event lists, migration review queues, and future provider capability matrices. | Empty, loading, paginated, sortable where useful, and inline warning state for partial failures. | Avoid dense grid layouts for core search results; reserve tables for operational data where scanning matters more than imagery. |
| Cards | Search results, timeline capture summaries, provider mode selections, and first-run decision tiles. | Default, hovered, selected, loading skeleton, processing, warning, and error. | Cards should consistently expose screenshot thumbnail, timestamp, app or window metadata, OCR excerpt, and action affordances when the underlying data exists. |
| Dialogs | Export destination confirmation, credential clear confirmation, quit while processing, and high-risk delete operations. | Open, submitting, blocked by validation, destructive, and recovery-failure. | Dialogs are for focused decisions only. Multi-step flows such as onboarding or full export configuration belong in routed panels instead. |
| Toasts | Low-friction feedback for saved settings, restarted services, completed tests, and transient recoverable errors. | Success, info, warning, error, and action-required. | Toasts must never be the only place a serious failure is reported. Any recorder stoppage, migration failure, or provider outage also appears in persistent shell status. |
| Destructive confirmations | Delete history, clear provider credentials, remove exports, and disable startup capture defaults. | Scope review, typed acknowledgement for bulk deletes, final confirm, job-started, and completion summary. | Show human-readable scope before confirm, including date range, item count estimate, and whether screenshots, text, embeddings, or secrets are affected. |

### Shared React/TypeScript Shell

- Layout composition: Use a persistent left navigation rail on desktop widths, a condensed top bar plus drawer on smaller widths, and a shell header that always shows recorder state, service health, and the active storage profile.
- Route ownership: Top-level routes map to Search, Timeline, Recorder, AI Setup, Settings, and Privacy/Data Management. Each route can own sub-navigation, but cross-route state such as recorder health and job banners is managed once at the app shell level.
- Data access pattern: Use route loaders or service hooks that fetch from the Python API and subscribe to the status stream for live updates. Mutations are optimistic only for low-risk preference toggles; recorder state changes, delete jobs, and migration operations wait for confirmed service acknowledgement.
- Visual system baseline: Favor a desktop-native, low-distraction shell with dense information panels, subdued chroma, clear status color semantics, and large screenshot surfaces. The design system should standardize spacing, elevation, keyboard focus treatment, and empty-state illustrations before deeper feature work begins.
- Accessibility expectations: Keyboard navigation, visible focus rings, screen-reader labels for status chips and icon-only controls, reduced-motion support, and sufficient color contrast are required from the first shell implementation.

## Recorder Experience

### Recorder Control Surface
- Start/Stop actions: Provide a primary `Pause Recorder` or `Resume Recorder` action in both the Recorder route and the tray menu. Stopping is a soft pause, not a destructive shutdown, and it requires only a lightweight confirmation when background OCR or export work is still active.
- Current status display: Always show `Running`, `Paused`, `Starting`, `Stopping`, `Degraded`, `Needs Attention`, or `Migrating` as a labeled status chip. Pair it with last capture time, configured interval, backlog depth, and the currently active processing mode (`Local only`, `Local plus AI`, or `AI disabled`).
- Error-state handling: If capture fails because of permissions, full disk, service crash, or OCR backlog overflow, show a persistent inline alert with the fault category, the impact on capture or search, and the next best recovery action. Users must be able to copy a local diagnostics bundle path from the same surface.
- Active capture indicator text: Use plain language strings such as `Capturing every 10 seconds`, `Paused by you`, `Waiting for screen activity`, `Finishing queued OCR`, or `Service restarting after failure` so the status is understandable without reading logs.

### Capture Frequency Controls

| Option | Validation Rules | Persistence Rules | Notes |
|--------|------------------|-------------------|-------|
| 10 seconds | Always available. Reject changes while migration is active. | Save immediately to recorder settings and push to the running service when healthy. | This is the recommended default for users who want high recall fidelity on capable hardware. |
| 1 minute | Always available. Reject changes while migration is active. | Save immediately and apply without requiring a recorder restart. | Good default for lower-power devices or privacy-conscious users who want fewer captures. |
| 5 minutes | Always available. Reject changes while migration is active. | Save immediately and apply without requiring a recorder restart. | Intended for very low-overhead capture. The UI should warn that sparse intervals may miss short tasks. |
| Custom value | Accept integer seconds only, minimum `10`, maximum `3600`, with inline validation and a helper note when the value is above `300`. | Persist the normalized value in settings, clamp unsupported values in the service, and echo the final accepted interval back to the renderer. | Present as `Custom` plus numeric input. If the service clamps the value because of platform limits, show the effective interval immediately. |

## First-Run and Returning-User UX

- First-run entry experience: After the main process confirms service readiness, open a focused onboarding flow instead of dropping directly into the full app. The first-run path covers storage location summary, migration status if a legacy database is present, capture permission expectations, and an explicit choice to start with default local-only settings.
- Returning-user entry experience: Returning users land in the last visited route after the shell hydrates current service and recorder status. If the service detects a migration issue, unresolved asset import, or failed restart recovery, the app opens the relevant review panel first and then returns the user to their last route once dismissed.
- AI-optional onboarding path: Onboarding includes a clear branch that says the app is fully usable without AI providers. The recommended path is `Start with local capture and search`, while a secondary branch opens AI Setup for users who want semantic enrichment or model-backed retrieval enhancements.
- Non-blocking default capture/search path: If the user skips AI setup, OpenRecall immediately enables the local timeline, lexical search, and recorder controls using the built-in service capabilities. Any later AI connection is additive and cannot block baseline browsing or capture.

## Settings Information Architecture

| Section | Audience | Contents | Destructive Actions Present? |
|---------|----------|----------|------------------------------|
| Routine settings | All users who want to adjust everyday app behavior without touching data lifetime or provider internals. | Theme, launch-on-login, close-to-tray behavior, default landing route, capture interval, monitor scope, idle pause, storage-folder reveal, and local diagnostics verbosity. | No. Routine settings may contain toggles that affect runtime behavior, but no irreversible delete or credential-clear actions appear here. |
| Advanced provider setup | Users who intentionally enable local or remote AI integrations. | Provider mode, endpoint URL, model names, trust warnings, test actions, capability status, timeout tuning, and secret replace or clear entry points. | Limited. Secret clear actions are available here but require a dedicated confirmation dialog and never appear inline beside routine settings. |
| Privacy/Data Management | Users handling retention, deletion, export, migration review, or trust-sensitive data changes. | Retention defaults, delete jobs, export jobs, unresolved migration assets, storage usage, diagnostics export, and provider credential removal summaries. | Yes. This is the only place where bulk destructive actions, export jobs, and credential-clearing workflows are grouped. |

## Desktop Lifecycle Behaviors

- Tray/background behavior: The app always creates a tray icon after successful main-process startup. The tray menu shows recorder status, quick pause or resume, `Open OpenRecall`, `Open Recorder`, `Run Diagnostics Export`, and `Quit`. If the main window is closed, the tray remains active while the service keeps capturing according to the saved policy.
- Close-to-tray vs quit: Closing the main window minimizes to tray by default after the first-run flow explains that behavior. `Quit` is only available from the tray, app menu, or an explicit in-app action, and it triggers the supervised shutdown sequence defined in Phase 01 so capture intake drains safely before the service exits.
- App startup state: On login or manual launch, Electron main starts first, then the Python service, then the renderer. If `launch_on_login` is enabled, the app starts hidden to tray unless the previous session ended in a blocking error or incomplete migration state. The recorder respects the persisted `auto-start recorder` preference rather than always beginning capture unconditionally.
- Service health visibility: Show service health in three places: a persistent shell status chip, the Recorder route detail panel, and the tray tooltip or menu label. Health categories must distinguish capture, OCR, embeddings, provider connectivity, migration, and storage capacity so the user can see what is degraded.
- Offline or degraded-mode behavior: Because OpenRecall is local-first, network unavailability should affect only remote provider features. Search, timeline, recorder controls, and privacy management continue to work locally. If OCR or embeddings are temporarily degraded, the UI clearly marks which features are delayed, which data is still available, and whether backlog processing will resume automatically.

## Diagnostics

- Local telemetry/logging policy: Phase 02 approves local-only diagnostics and explicitly rejects default remote telemetry. The app records structured local logs for service startup, recorder state changes, queue depth, migration progress, provider tests, and major error categories. Users can opt into a higher local verbosity level from settings when troubleshooting.
- Privacy constraints: Diagnostics must exclude screenshot pixels, OCR text, embeddings, search queries, and raw provider secrets. File paths should be minimized to relative identifiers where possible, and exported diagnostic bundles require explicit user action from the desktop UI.
- Retention policy: Keep rolling local logs for a short operational window, defaulting to 14 days or a small capped file size, whichever comes first. Diagnostic export bundles are created on demand, stored locally in a user-chosen destination, and never transmitted automatically.

## Open Questions

- Whether the first Windows release should include native notification center alerts for recorder failures, or rely on shell banners plus tray-state changes until notification fatigue rules are better understood.
- Whether the compact Recorder route should eventually support an always-on-top mini controller for power users, or if tray controls are sufficient for the first desktop release.

## Acceptance Criteria

- The desktop shell and renderer boundaries are clear
- Navigation and major UX surfaces are fully mapped
- Recorder and settings interactions are specified with validation and state behavior
- Tray, startup, and service-health behaviors are explicit enough for implementation
