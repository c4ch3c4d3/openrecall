# OPENRECALL-DESKTOP-05 - Search, Timeline, and Retrieval Rework

## Scope

This document defines the data model, indexing strategy, hybrid retrieval pipeline, timeline UX, and performance requirements for the redesigned OpenRecall search experience.

## Required Outcomes

- [x] Define a hybrid retrieval model with structured filters, lexical search, semantic search, and reranking
- [x] Extend stored records to cover screenshots, OCR text, timestamps, app/window metadata, embeddings, and derived summaries
- [x] Specify richer search filters and user-oriented query handling
- [x] Define fuzzy matching and typo tolerance
- [x] Specify semantic retrieval and hybrid ranking behavior
- [x] Define reranking behavior
- [x] Design the dedicated timeline UI and drill-down flow
- [x] Add explainability signals for ranked results
- [x] Define long-running performance, indexing, and reindexing behavior

## Retrieval Objectives

- Replace the current "load every row and compare every embedding in memory" search path with a staged retrieval pipeline that can serve large local datasets without blocking capture.
- Keep search and timeline useful when AI is disabled, indexing is catching up, or provider-backed enrichment is partially unavailable.
- Make every ranked result explainable through snippets, metadata matches, time cues, and clear labeling of which ranking stages were used.
- Preserve a single canonical local record for each capture while allowing OCR, embeddings, summaries, and future enrichments to evolve independently.

## Data Model

| Field Group | Fields | Query Use | Storage Notes |
|-------------|--------|-----------|---------------|
| Capture metadata | `capture_id`, `captured_at`, `capture_day`, `capture_hour_bucket`, `relative_screenshot_path`, `monitor_id`, `width`, `height`, `content_hash`, `session_id`, `processing_state`, `deleted_at` | Timeline ordering, date filters, duplicate handling, drill-down navigation, export and deletion scope | Canonical row in `captures`. `processing_state` distinguishes `queued`, `ocr_ready`, `search_ready`, `summary_ready`, `failed`, and `deleted_soft`. Screenshot paths remain relative to the managed media root approved in Phase 01. |
| OCR text | `raw_text`, `normalized_text`, `primary_excerpt`, `language_code`, `token_count`, `ocr_engine_version`, `ocr_completed_at` | Lexical matching, snippet extraction, typo-tolerant candidate expansion, language-aware display | Stored in `capture_text` keyed by `capture_id` and mirrored into an FTS5 table over normalized text plus denormalized app and title fields needed for ranking. Keep raw OCR for export and audits; search reads normalized text. |
| App/window metadata | `app_name`, `window_title`, `executable_name`, `window_class`, `url_hint`, `is_browser_surface`, `focus_confidence` | App filters, title search, browser-session recall, exact metadata boosts, explainability | Stored in `capture_context`. `url_hint` is optional and must stay local-only. Missing fields are valid; search cannot assume browser metadata exists. |
| Embeddings | `embedding_profile_id`, `vector_dimension`, `vector_blob`, `embedding_source`, `provider_mode`, `model_id`, `embedded_at`, `embedding_status` | Semantic search, hybrid retrieval, profile-aware reindexing, degradation messaging | Stored in `capture_embeddings`. Multiple profiles may coexist during migration or provider changes, but only one profile is active for default ranking at a time. Missing vectors do not block lexical results. |
| Derived summaries | `summary_text`, `summary_source`, `summary_status`, `summary_version`, `tag_list`, `entity_hints`, `importance_score`, `last_enriched_at` | Timeline previews, semantic boosting, explainability chips, future export surfaces | Stored in `capture_derivatives`. Provider-generated summaries remain optional; when absent, UI falls back to OCR excerpts. Summaries must never be the only searchable text representation. |

### Approved Record Model

- `captures` is the canonical table for timeline identity, file ownership, and lifecycle state.
- OCR, embeddings, and summaries are append-only or versioned side tables so the service can retry or rebuild them without rewriting screenshot assets.
- Search never infers screenshot filenames from timestamps. Every result and timeline item resolves media through the stored relative path.
- Soft-delete markers remain visible to maintenance jobs until the hard-delete semantics in Phase 05 finalize removal and compaction rules.

## Indexing Strategy

- Lexical index: Use SQLite FTS5 as the day-1 lexical engine over `normalized_text`, `app_name`, `window_title`, and optional `summary_text`. Configure `unicode61` tokenization with prefix indexes for `2`, `3`, and `4` character prefixes so partial-term recall and incremental typing remain fast. Rank lexical candidates with BM25 plus exact-match boosts on app and title fields.
- Semantic index: Use the active embedding profile from `capture_embeddings` as the source for semantic candidate generation. For the first desktop release, semantic retrieval may use brute-force cosine similarity over the filtered candidate pool or a rebuildable auxiliary vector structure, but any auxiliary index must be derived entirely from SQLite embeddings and be safe to discard and rebuild.
- Metadata filters: Apply hard filters before semantic scoring whenever possible: date range, app names, window-title contains, monitor, processing state, and provider-enrichment availability. Hard filters reduce semantic candidate size and are part of the explainability output.
- Reindex triggers: Trigger scoped reindex or re-embed work on schema migration, OCR normalizer version changes, FTS tokenizer changes, embedding profile switches, summary-version changes, manual `Rebuild Search Index` requests, and corruption recovery.
- Backfill strategy: New imports from the legacy schema create canonical `captures` rows first, then enqueue OCR normalization, FTS population, embedding verification or regeneration, and summary derivation as independent durable jobs. Search remains available during backfill with partial-result labeling such as `Text ready, semantic index pending`.

### Indexing Rules

- Capture ingestion writes the minimal canonical record first, then enqueues downstream search work.
- OCR completion updates FTS in the same transaction that marks the text row `ready`.
- Embedding generation is asynchronous and version-aware. Switching embedding profiles must not wipe older vectors until the new profile reaches a usable coverage threshold.
- Reindex jobs are resumable and chunked by day or capture-id range so crashes do not require full restarts.

## Query and Filter Model

| Query Type | User Example | Required Inputs | Ranking Signals |
|------------|--------------|-----------------|-----------------|
| Time range | `last Tuesday afternoon` | Resolved start and end timestamps, timezone, optional daypart | Temporal proximity, exact bucket overlap, recent anchor preference when the query implies recency |
| App/window title | `that Slack thread with the release checklist` | App tokens, title tokens, optional exact-app filter | Exact app match, title phrase match, repeated-session continuity, recency within the filtered scope |
| Raw OCR text | `postgres password reset link` | OCR terms or phrase, optional quote mode, optional typo tolerance flag | Exact phrase hit, token coverage, term rarity, snippet density, proximity inside OCR text |
| User-oriented recall query | `the chart I was looking at before the budget meeting` | Natural-language query, optional structured filters, active embedding profile if available | Lexical score, semantic similarity, rerank score, time cues, app/title metadata, summary or tag alignment |

### Supported Filters

- Time: absolute range, relative range (`today`, `yesterday`, `last 7 days`), daypart (`morning`, `afternoon`, `evening`), and exact timestamp jump.
- Source context: app name, executable, window-title contains, browser-only surfaces, monitor id, and session cluster.
- Processing state: `ready`, `text pending`, `semantic pending`, `summary pending`, `failed`, and `import review needed`.
- Result shape: only items with screenshots, only items with summaries, only items with provider enrichment, and screenshot aspect ratio buckets for UI layout stability.

### User-Oriented Query Handling

- The search box accepts a single free-text query and optional structured filters from the surrounding UI instead of forcing users to learn query syntax.
- The service performs lightweight query understanding for time phrases, quoted exact phrases, and obvious app-name aliases, then returns the parsed interpretation so the renderer can show removable filter chips.
- When the parser is uncertain, it keeps the original text as a general recall query and avoids silently discarding terms.
- Empty search with filters is valid and returns the filtered timeline-like result set ordered by timestamp.

## Retrieval Pipeline

### Stage 0: Query Normalization and Intent Hints

1. Normalize whitespace, punctuation, and case for matching while preserving the original query for display.
2. Extract explicit quoted phrases, obvious time phrases, and app aliases from a maintained local synonym table.
3. Build a `query_plan` object containing hard filters, lexical terms, semantic eligibility, and whether reranking should run.

### Lexical and Fuzzy Retrieval

- OCR keyword matching: Query the FTS5 index first using exact terms, quoted phrases, and prefix expansions. Favor OCR hits with dense term coverage in a short text span over sparse hits across noisy OCR.
- App/window metadata matching: Run metadata lookups alongside FTS so an exact app name or title phrase can boost or rescue relevant items even when OCR is weak.
- Typo tolerance: If the lexical pass returns too few high-confidence hits, run a bounded fuzzy expansion pass against a maintained dictionary of common OCR tokens, app names, and recent title tokens. Use Damerau-Levenshtein distance `<= 1` for short terms and `<= 2` for longer terms, capped to a small expansion set per token. Never broaden quoted exact phrases.
- Snippet extraction: Extract the shortest high-signal excerpt containing the matched terms, preserving OCR line order where possible. If OCR snippets are poor, fall back to title or summary snippets with clear labeling.

### Semantic Retrieval

- Embedding source: Use the active embedding profile approved in Phase 04: built-in local embeddings by default, provider-backed embeddings only when that capability is validated and healthy.
- Query embedding behavior: Generate one query embedding per search request when the query is semantic-eligible and the active embedding provider is available. Cache only short-lived in-memory query vectors for the session; do not persist them to disk or logs.
- Candidate generation: Compute semantic similarity only across records that satisfy hard filters and have a ready embedding for the active profile. If no semantic profile is ready, return lexical-only results and label the mode accordingly. Initial semantic candidate pool target is top `200` filtered captures before reranking.

### Hybrid Ranking and Reranking

- Initial candidate merge: Union lexical and semantic candidate ids, deduplicate by `capture_id`, and annotate each candidate with which retrieval stages produced it.
- Weighting strategy: Day-1 ranking uses a weighted blend where lexical evidence stays dominant for exact-text queries and semantic evidence rises for natural-language recall queries. The default weighting target is approximately `0.55 lexical`, `0.30 semantic`, `0.10 metadata boosts`, and `0.05 recency or session continuity`, with the semantic share dropped to `0` when embeddings are unavailable.
- Reranking inputs: Send only the top `50` hybrid candidates into the rerank stage when a validated reranker exists and the query is not a pure metadata filter. Reranker input includes the original user query, OCR excerpt, title, app name, summary text when available, and timestamp cues, but never raw screenshot bytes unless a future capability explicitly requires it.
- Final score explanation: The response returns `retrieval_mode`, `lexical_score`, `semantic_score`, `rerank_score` when present, plus structured match reasons such as `phrase hit in OCR`, `app matched Slack`, `semantic similarity to summary`, or `same session as top hit`.

### Ranking Guarantees

- Exact phrase hits in OCR or title outrank semantic-only matches for the same filtered scope.
- Reranking may reorder candidates inside the retrieved pool but may not introduce new candidates absent from lexical or semantic retrieval.
- When reranking is unavailable or times out, the service returns the hybrid ranking immediately and labels the skipped stage instead of retrying indefinitely.
- Result pages must be stable across pagination by sorting ties on `captured_at DESC, capture_id DESC`.

## Timeline Experience

### Timeline UI

- Chronological view structure: The dedicated Timeline route uses a reverse-chronological, day-grouped feed with sticky date headers and optional session clustering inside each day. Default density is a mixed card grid that emphasizes screenshot thumbnails without hiding timestamps or app names.
- Image/text pairing: Every timeline card pairs the screenshot thumbnail with timestamp, app or window metadata, OCR excerpt, and a processing badge such as `Text pending` or `Summary ready`. If OCR is unavailable, the card still renders the image and metadata rather than collapsing.
- Fast navigation: Provide a date scrubber, keyboard page navigation, jump-to-now, jump-to-date, and quick relative ranges (`Today`, `Yesterday`, `Last 7 days`, `This month`). The route must remember scroll position when the user opens and closes a detail panel.
- Time filters: Timeline supports the same date, app, monitor, and processing-state filters as Search, but defaults to chronological ordering instead of ranked ordering.
- Drill-down details: Selecting a card opens a detail panel or full-view route with the full screenshot, neighboring captures, OCR text, derived summary, explainability metadata, and actions to `Search similar`, `Jump to surrounding timeline`, or `Copy timestamp`.

### Recall and Forensic Review Support

- Quick jumps to dates/times: The UI supports direct date entry, hour-level jumps, and keyboard shortcuts for previous or next result within the current filter scope.
- Adjacent-entry browsing: The detail panel always exposes the immediate previous and next captures in the filtered timeline so users can reconstruct activity sequences without re-running a search.
- Evidence-oriented use cases: Timeline and detail views show exact timestamps, app or title metadata, monitor id when relevant, and processing-state notes so users can validate what was captured and what remains pending.
- Casual recall use cases: Search results can pivot into timeline context around a hit, and timeline cards can initiate a related search from the visible OCR snippet or app name.

### Search-to-Timeline Flow

1. User runs a search from the Search route.
2. Ranked results show explainability chips and a `View in Timeline` action.
3. Choosing that action opens Timeline centered on the result's timestamp with surrounding captures preloaded.
4. Returning to Search preserves the original query, filters, and scroll position.

## Explainability Signals

- Matched text snippets: Show the exact OCR or summary snippet that matched, with matched tokens highlighted and the source labeled as `OCR`, `Title`, or `Summary`.
- Matched app names: Surface app or executable matches as explicit chips such as `Matched app: Slack` or `Matched title: Budget Review - Chrome`.
- Timestamp cues: Add cues such as `Matched time filter: last Tuesday afternoon`, `Captured 8 minutes before top result`, or `Same session as a strong hit`.
- Semantic-match explanation: When semantic ranking or reranking contributes materially, show short labels such as `Conceptually similar to your query` or `Summary matched the described chart review`. Do not fabricate precise semantic reasons that the model cannot support.

### Explainability Contract

- Every result must include at least one human-readable match reason.
- Search responses expose both machine-readable reason codes and renderer-ready labels so UI surfaces remain consistent.
- The renderer must distinguish `Lexical only`, `Lexical + semantic`, and `Lexical + semantic + rerank` result sets.
- If a result is included primarily because of timeline adjacency or session continuity, that reason must never masquerade as a direct text hit.

## Performance and Long-Running Behavior

- Expected dataset size: Design for at least `250,000` captures, multi-year local history, and day ranges with thousands of screenshots without full-table scans in the interactive path.
- Background indexing behavior: Capture intake remains the highest-priority write path. OCR normalization, FTS updates, embeddings, summaries, and auxiliary index maintenance run as bounded background jobs with separate queues and health counters.
- Reindexing behavior: Manual or automatic reindex runs operate in resumable chunks, expose progress by phase (`text`, `fts`, `embeddings`, `summaries`), and allow search to keep serving partial results from the last complete index state.
- Capture non-interference rules: Search and indexing must yield to active capture on CPU, disk, and database contention. If contention exceeds the soft budget, defer semantic work first, then summary derivation, while keeping canonical capture writes and FTS updates current.
- Resource limits: Target search p95 under `400 ms` for lexical-only queries on warm caches and under `1500 ms` when semantic retrieval plus reranking are active on the default result size. Cap reranking concurrency to avoid starving OCR workers. Keep reindex worker memory bounded and below the Phase 01 Python-service soft cap.

### Operational Budgets

- Lexical candidate cap: `500` candidates before merge.
- Semantic candidate cap: `200` candidates before merge.
- Rerank cap: `50` candidates per request.
- Default page size: `50` search results, `100` timeline cards.
- Thumbnail generation: Precompute or cache small thumbnails asynchronously; do not resize full screenshots on every list request.

### Failure and Recovery Rules

- If FTS is unavailable or rebuilding, fall back to timestamp and metadata filtering with a clear `Search index rebuilding` notice.
- If embeddings are missing or stale for the active profile, exclude them from semantic ranking and emit `semantic pending` counts in health and search metadata.
- Corrupt screenshot files mark the capture as review-needed but do not block surrounding search or timeline results from rendering.
- Repeated indexing failures move the affected items into a visible failed-job queue rather than looping forever.

## Approved Decisions

- SQLite FTS5 is the required lexical engine for the first desktop release.
- Hybrid retrieval begins with lexical plus stored embeddings, while any optional ANN layer remains an implementation detail that must be fully rebuildable from SQLite records.
- Timeline remains fully usable without AI providers, using screenshots, metadata, and OCR excerpts as the baseline presentation.
- Reranking is optional and capability-scoped. When unavailable, the service returns the hybrid ranking with explicit stage labeling rather than treating it as an error.
- Explainability is a product requirement, not a later enhancement. Search results and timeline drill-downs must disclose why an item appeared.

## Open Questions

- Whether the first desktop release should store browser URL hints when they are locally available, or defer that metadata until the privacy and export semantics in Phase 05 are finalized.
- Whether session clustering should be based only on temporal gaps and app continuity, or also incorporate semantic similarity once retrieval benchmarks exist.
- Whether provider-generated summaries should participate directly in FTS ranking by default, or remain a lower-weight auxiliary field until benchmarked against OCR-heavy datasets.

## Acceptance Criteria

- The storage and indexing strategy supports hybrid retrieval from day 1
- Search filters and timeline flows are clearly specified
- The ranking pipeline includes lexical, semantic, and reranking stages
- Explainability fields and degraded-mode labels are explicit enough for UI and API implementation
- Long-running performance behavior is documented well enough to guide implementation
