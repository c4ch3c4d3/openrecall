# OPENRECALL-DESKTOP-04 - AI Provider Configuration and Processing Pipeline

## Scope

This document defines how OpenRecall supports optional AI provider configuration, multimodal processing requirements, provider routing, fallback behavior, and provider-health reporting.

## Required Outcomes

- [x] Define provider modes for Disabled, Ollama, LM Studio, and remote OpenAI-compatible endpoints
- [x] Specify a dedicated AI setup flow that explains multimodal requirements clearly
- [x] Keep local model setup optional instead of mandatory
- [x] Define provider-specific configuration forms and validation
- [x] Specify backend abstractions for OCR, image understanding, embeddings, and reranking
- [x] Remove local-only assumptions from the AI contract while preserving the local-first product baseline
- [x] Define graceful fallback behavior when provider setup is missing or incomplete
- [x] Add provider health and last-success reporting
- [x] Document capability boundaries by provider

## Provider Mode Matrix

| Mode | Use Case | Required Fields | Optional Fields | Constraints |
|------|----------|-----------------|-----------------|-------------|
| Disabled | Default state for users who want local capture, local OCR, timeline, and lexical search without any model endpoint setup. | None. | Future provider draft values may remain saved but inactive. | Baseline product use must still work. Semantic enrichment, provider-backed vision, remote embeddings, and reranking stay off. |
| Ollama | User wants a local endpoint for vision, embeddings, or text generation with models managed outside OpenRecall. | Provider mode, endpoint URL, at least one model assignment for the enabled capabilities. | Timeout, retries, concurrency cap, TLS override for reverse-proxy use, and capability toggles. | Treat as local only when the host resolves to loopback. Capability test must confirm vision support before image understanding is enabled. |
| LM Studio | User wants a desktop-hosted OpenAI-compatible endpoint with local models and a familiar API shape. | Provider mode, endpoint URL, at least one model assignment for the enabled capabilities. | Optional API key if LM Studio is configured with auth, timeout, retries, and capability toggles. | Compatibility claims are not sufficient on their own. OpenRecall must probe for chat, embeddings, and image support because LM Studio model metadata can be incomplete. |
| Remote OpenAI-compatible | User wants to opt into a hosted endpoint for multimodal understanding, embeddings, or reranking. | Provider mode, endpoint URL, API key reference, at least one model assignment, and explicit remote-trust acknowledgement. | Organization or project id, timeout, retries, concurrency cap, custom headers, and cost warning thresholds. | Remote providers are optional integrations only. They can never become a prerequisite for capture, local browsing, local OCR, or delete or export flows. |

## Provider Strategy

- Baseline contract: OpenRecall remains local-first for capture, screenshot storage, OCR persistence, timeline browsing, and lexical search. The built-in Python OCR pipeline remains the default source of extracted text even when no provider is configured.
- Provider routing contract: Providers are additive workers for multimodal image understanding, alternate embedding generation, and reranking. The service routes work per capability instead of assuming one provider owns the full pipeline.
- Default mode: New installs and migrated users start in `Disabled` mode with the onboarding recommendation `Start with local capture and search`.
- Capability activation rule: A provider only becomes active for a capability after a successful capability-specific test. Saving configuration alone is not enough to turn on vision, embeddings, or reranking.
- Trust model: Loopback endpoints are treated as local integrations. Any endpoint outside `127.0.0.1`, `::1`, or `localhost` is labeled remote, requires explicit user acknowledgement, and is surfaced in diagnostics as network-dependent.

## AI Setup Flow

- Entry point: The primary entry is the `AI Setup` route defined in Phase 02. First-run onboarding links to it as a secondary branch, and the Search plus Recorder routes can deep-link into it when the user requests a disabled capability.
- User education about multimodal requirements: The first panel explains that OpenRecall already works without AI setup, then clarifies that provider-backed image understanding requires a multimodal vision-capable model, embeddings require a compatible embedding model, and reranking is optional. The copy must explicitly warn that many OpenAI-compatible endpoints are text-only even when they expose the same REST shape.
- Deferred setup path: Users can skip setup with a single `Continue with local capture and search` action. Skipping keeps provider mode at `Disabled`, suppresses repeated onboarding prompts, and leaves a passive `AI is optional` card in Search and Settings rather than a blocking banner.
- Success state: After a successful connection test, the UI shows the active mode, validated capabilities, last test time, and which product features are now enhanced. The navigation label changes from `AI Setup` to `AI Providers`, and the search experience can start using the enabled capabilities on newly queued work.
- Failure state: Failed tests keep the mode in a `Configured but inactive` state, preserve non-secret form values, and show the specific failure category: unreachable endpoint, auth failure, capability mismatch, timeout, or malformed response. The user can save the draft without activation, retry the test, or revert to `Disabled`.

### Setup Flow Stages

1. Choose mode: `Disabled`, `Ollama`, `LM Studio`, or `Remote OpenAI-compatible`.
2. Explain capability needs: show a short matrix for `Vision`, `Embeddings`, and `Reranking` with notes on what each capability changes in the product.
3. Enter configuration: render a provider-specific form with validation before any secret is stored.
4. Test connection: run `POST /v1/providers/test` for the requested capabilities and return a normalized capability report.
5. Confirm activation: enable only the capabilities that passed, keep failures disabled, and summarize fallback behavior before the user leaves the route.

## Provider Configuration Forms

| Field | Applies To | Validation | Persistence | Notes |
|-------|------------|------------|-------------|-------|
| Provider mode | All modes | Required enum value. Switching away from an active provider requires confirmation if it disables capabilities currently in use. | Non-secret provider settings in `settings.json`. | `Disabled` is a first-class saved state, not the absence of config. |
| Endpoint URL | Ollama, LM Studio, Remote OpenAI-compatible | Required valid URL for enabled providers. Reject empty values, unsupported schemes, and URLs with embedded credentials. Warn when host is not loopback. | Non-secret metadata in `settings.json`. | Prepopulate local defaults for Ollama and LM Studio, but still run validation and tests before activation. |
| Vision model | Ollama, LM Studio, Remote OpenAI-compatible | Required only when image understanding is enabled. Must be a non-empty model id accepted by the capability test response. | Non-secret metadata in `settings.json`. | This may differ from the embedding or reranking model. |
| Embedding model | Ollama, LM Studio, Remote OpenAI-compatible | Required only when provider-backed embeddings are enabled. Must produce the expected vector size recorded in the capability report. | Non-secret metadata in `settings.json`, plus vector profile version in SQLite metadata. | Changing this value queues a controlled re-embed job in a later phase. |
| Reranking model | Ollama, LM Studio, Remote OpenAI-compatible | Optional. If present, test must confirm the endpoint can score or rank candidate pairs. | Non-secret metadata in `settings.json`. | If omitted, retrieval falls back to lexical plus semantic ranking only. |
| Auth/API key | LM Studio when configured, Remote OpenAI-compatible | Required for remote endpoints unless the provider explicitly supports anonymous access. Never echo stored values back to the renderer. | Secret stored in OS keychain via Electron main; only a key reference is stored in settings. | Renderer may show `Stored`, `Missing`, or `Needs replacement`, but not the secret itself. |
| Timeout | Ollama, LM Studio, Remote OpenAI-compatible | Integer milliseconds, minimum `1000`, maximum `120000`. | Non-secret metadata in `settings.json`. | Separate connect and request timeouts are allowed internally, but the UI exposes one advanced timeout control initially. |
| Retry settings | Ollama, LM Studio, Remote OpenAI-compatible | Integer attempts `0-5` and backoff profile `none`, `linear`, or `exponential`. | Non-secret metadata in `settings.json`. | Retries apply to test and processing calls, but not to auth failures or capability mismatches. |
| Capability toggles | Ollama, LM Studio, Remote OpenAI-compatible | Allow only `image_understanding`, `provider_embeddings`, and `reranking`. Reject enabling a capability that has not passed testing. | Non-secret metadata in `settings.json`. | Built-in OCR remains separate and cannot be disabled from this form. |
| Remote trust acknowledgement | Remote OpenAI-compatible and any non-loopback URL | Required boolean acknowledgement before activation. | Non-secret metadata in `settings.json`. | Copy must explain that screenshots or derived text may be sent to the remote endpoint for enabled capabilities. |
| Test connection action | Ollama, LM Studio, Remote OpenAI-compatible | Disabled until the required fields for the selected capabilities are valid. | Test results persisted as health metadata, not as authoritative capability config. | Returns a normalized report with per-capability pass or fail results, latency, and server identity when available. |

### Form Behavior Rules

- Provider-specific rendering: `Disabled` shows only explanatory copy and prior-health history. Local modes show endpoint plus model fields, while remote mode adds trust and credential sections.
- Save semantics: `Save draft` persists non-secret form state without activating provider-backed processing. `Save and test` persists the draft, writes secrets through Electron main if present, and immediately runs the capability probe.
- Secret handling: Replacing or clearing secrets always uses a dedicated modal. Clearing credentials automatically disables only the capabilities that require that secret.
- Validation timing: Structural validation runs inline on change. Capability validation runs only through the explicit test action so the user understands why a model assignment was accepted or rejected.

## Backend Provider Abstractions

| Capability | Abstraction | Provider-Specific Behavior | Fallback |
|------------|-------------|----------------------------|----------|
| OCR | `OcrEngine` owned by the Python service. Default implementation is the built-in local OCR pipeline. | Providers do not replace baseline OCR in the first desktop release. Provider-assisted OCR can be explored later behind a separate adapter without changing the capture contract. | Capture continues with local OCR. If local OCR is degraded, screenshots are still stored and queued for retry, and timeline cards show `Text pending`. |
| Image understanding | `VisionAnalyzer` with a normalized request and response shape for screenshot-to-summary or screenshot-to-tags tasks. | Ollama and LM Studio use OpenAI-style chat or vision adapters when they can accept image input. Remote endpoints may require stricter token or image-size limits and can return richer summaries. | If unavailable, OpenRecall omits AI-generated summaries and tags. Timeline, screenshot viewing, and lexical search remain available from OCR text only. |
| Embeddings | `EmbeddingProvider` that returns vector payload plus profile metadata `{ provider_mode, model_id, dimension, version }`. | Local built-in embeddings remain the default profile for baseline semantic search. Provider-backed embeddings are opt-in and must declare a stable vector dimension before activation. | Search falls back to lexical retrieval plus whatever embeddings already exist for the active profile. Missing embeddings queue for later processing instead of blocking results. |
| Reranking | `Reranker` that scores a query against already retrieved candidates and returns ordered ids plus reason metadata. | Remote endpoints may use hosted rerank APIs or chat-based ranking prompts. Local endpoints may support only prompt-based ranking with higher latency and lower consistency. | If unavailable, retrieval returns lexical plus semantic ranking without the rerank stage and labels the result set accordingly. |

### Provider Router Rules

- The Python service owns provider selection through a `ProviderRouter` that resolves a capability-specific adapter from the saved provider mode and validated capability report.
- Each capability keeps separate health, concurrency, timeout, and last-success metadata so one failing feature does not disable the entire provider stack.
- Queue workers request capability handles from the router at execution time, not at app startup, so changing provider settings does not require a full service restart.
- The router must return deterministic failure categories: `disabled`, `not_configured`, `auth_missing`, `capability_missing`, `endpoint_unreachable`, `timeout`, `rate_limited`, `response_invalid`, or `internal_error`.

## Graceful Degradation Rules

- Behavior when no provider is configured: The app stays in `Disabled` mode, continues capture, local OCR, timeline browsing, and lexical search, and labels semantic enhancements as unavailable rather than broken.
- Behavior when the provider is misconfigured: Preserve the draft config, disable only the affected capabilities, surface the validation or auth failure in `AI Providers`, and continue using baseline local processing for everything else.
- Behavior when the provider is unavailable: Keep queued provider-backed jobs in a retryable state with bounded backoff, mark search results with the last successful enrichment time, and avoid blocking capture or timeline rendering on provider recovery.
- Behavior when the endpoint is compatible but not multimodal: Accept the endpoint only for the capabilities it actually passed. A text-only endpoint may remain active for embeddings or reranking while image understanding stays disabled with a clear explanation.
- Capture/search behavior that must remain available regardless: Screenshot capture, screenshot persistence, recorder controls, migration, deletion, export, timeline browsing, OCR text storage, and lexical search must remain available even when every provider-backed capability is disabled or failing.

### Degradation Messaging Rules

- Shell status should distinguish `AI Disabled`, `AI Configured`, `AI Degraded`, and `AI Unavailable` instead of collapsing everything into a single warning.
- Search must label whether a result used `Lexical only`, `Lexical + local embeddings`, or `Lexical + provider enhancements` so degraded provider states are visible but not alarming.
- Recorder status should never show the provider as a hard blocker unless the user explicitly enabled a provider-only feature that is now failing.

## Provider Health and Status Reporting

| Signal | Source | User-Facing Message | Recovery Action |
|--------|--------|---------------------|-----------------|
| Configuration status | `GET /v1/settings/providers` sanitized config plus the most recent test result | `AI disabled`, `Configured draft`, `Active for embeddings`, `Active for vision and embeddings`, or `Configuration needs attention`. | Open `AI Providers`, edit the draft, or switch back to `Disabled`. |
| Last successful processing time | Per-capability health metadata emitted through `GET /v1/health` and `/v1/status/stream` | `Last embedding success 3 minutes ago` or `Vision has not succeeded yet`. | Retry the test, inspect diagnostics, or keep working with baseline local features. |
| Endpoint failure | Runtime error telemetry from provider workers with normalized categories | `Provider endpoint unreachable`, `Request timed out`, `Authentication failed`, or `Rate limited`. | Retry automatically within bounded limits, then prompt the user to test again or adjust endpoint, secret, or timeout settings. |
| Capability mismatch | Result of `POST /v1/providers/test` and runtime schema checks | `Connected endpoint does not support image input` or `Embedding model returned an unexpected vector size`. | Disable the failing capability, keep passed capabilities active, and prompt the user to pick a compatible model. |

### Health Contract

- Health is tracked per capability, not per provider mode only.
- `last_success_at`, `last_failure_at`, `failure_count`, `failure_category`, and `validated_capabilities` must be exposed to the renderer in sanitized form.
- Provider health signals appear in the AI route, the Recorder route health section, and local diagnostics export, but they must not include raw prompts, OCR text, screenshot bytes, or secrets.

## Capability Boundaries

- Vision/image understanding requirements: Image understanding requires an endpoint and model that can accept screenshot image payloads and return structured text output within OpenRecall's timeout budget. A generic chat model without image support does not satisfy this requirement even if the endpoint claims API compatibility.
- Text-only limitations: Text-only providers may still be used for embeddings or reranking when validated, but they cannot generate screenshot summaries, visual tags, or other image-derived metadata. The UI must present these limits as capability-specific, not as a full provider failure.
- Remote endpoint compatibility caveats: OpenAI-compatible only guarantees a similar transport contract, not matching feature support. Providers may differ on model naming, image payload shape, embedding dimensions, token limits, auth headers, and streaming semantics, so capability probing is mandatory.
- Unsupported provider behavior: OpenRecall does not support arbitrary script hooks, provider-written files, hidden background downloads initiated by the renderer, or silently sending screenshots to remote services. Any provider that cannot be represented through the normalized capability contracts remains unsupported in the first desktop release.

## Approved Decisions

- Built-in local OCR remains the default text extraction path in every provider mode, including `Disabled`.
- Provider-backed AI is capability-scoped. A single endpoint can be active for embeddings but inactive for vision or reranking.
- `Disabled` is the default and recommended setup path for first-run users who want immediate local capture and search.
- Remote OpenAI-compatible endpoints are supported as explicit opt-in integrations, but they do not alter the product's local-first storage and privacy baseline.
- Provider health must expose capability validation and last-success metadata so the UI can explain what is working, what is degraded, and what has never been enabled.

## Open Questions

- Whether the first desktop release should allow provider-backed embeddings to replace the built-in local embedding profile globally, or require them to stay as a secondary profile until Phase 04 retrieval benchmarks are complete.
- Whether OpenRecall should persist provider-generated screenshot summaries immediately in the main capture tables, or stage them in derived metadata tables until the long-term search and export semantics are finalized in Phases 04 and 05.

## Acceptance Criteria

- AI remains optional for meaningful product use
- Provider forms, validation, and health states are fully defined
- Backend abstraction boundaries are clear enough to implement without UI coupling
- Graceful fallback behavior is documented for incomplete or failed provider setups
