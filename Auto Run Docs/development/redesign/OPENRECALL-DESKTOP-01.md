# OPENRECALL-DESKTOP-01 - Redesign Program Overview

## Purpose

This document is the master index for the OpenRecall desktop redesign. It tracks the redesign scope, cross-phase principles, sequencing, and completion status for the detailed phase documents.

## Objective

Use this document as the orchestration entrypoint for the desktop redesign. Its tasks should drive execution of the remaining desktop phase files in order, and the redesign is not complete until `OPENRECALL-DESKTOP-02.md` through `OPENRECALL-DESKTOP-06.md` are all filled out and their acceptance criteria are satisfied.

## Instructions

1. Read `Improvement Plan.md` for the original redesign requirements
2. Use this file to determine sequencing and cross-phase dependencies
3. Run `OPENRECALL-DESKTOP-02.md`
4. Run `OPENRECALL-DESKTOP-03.md`
5. Run `OPENRECALL-DESKTOP-04.md`
6. Run `OPENRECALL-DESKTOP-05.md`
7. Run `OPENRECALL-DESKTOP-06.md`
8. Return to this file and update the deliverable status table and status notes

## Task

- [x] **Run `OPENRECALL-DESKTOP-02.md`:** Completed the Architecture and Foundations document with a current-state audit, target multi-process architecture, service surface, settings model, persistence/migration plan, efficiency rules, and security/privacy baselines.
- [x] **Run `OPENRECALL-DESKTOP-03.md`:** Completed the desktop shell and core UX framework with approved Electron shell boundaries, preload constraints, development and packaging workflow, navigation model, recorder controls, settings IA, tray and startup behavior, and local-only diagnostics policy.
- [x] **Run `OPENRECALL-DESKTOP-04.md`:** Completed the AI Provider Configuration and Processing Pipeline document with provider mode definitions, optional AI setup flow, provider-specific validation, capability-scoped backend abstractions, degradation rules, and provider health reporting.
- [x] **Run `OPENRECALL-DESKTOP-05.md`:** Completed the Search, Timeline, and Retrieval Rework document with the approved hybrid record model, FTS5 plus embeddings retrieval pipeline, filter and typo-tolerance rules, explainability contract, timeline drill-down flow, and long-running indexing and reindexing behavior.
- [x] **Run `OPENRECALL-DESKTOP-06.md`:** Completed the Privacy Controls, Deletion, Export, and Hardening document with the approved Privacy/Data Management IA, delete-all and time-range deletion semantics, operations-log export contract, migration and backfill rules, automated test gates, packaging and hardening requirements, and end-to-end release acceptance checks.
- [x] **Update program status in `OPENRECALL-DESKTOP-01.md`:** Updated the deliverable map to show all phase documents as complete, added status notes summarizing the approved retrieval and privacy/export decisions, and narrowed the remaining program-level open questions to the unresolved rollout items.

## Completion Rule

This document is only complete when all task items above are checked and the deliverable map shows `OPENRECALL-DESKTOP-02.md` through `OPENRECALL-DESKTOP-06.md` as complete or approved.

## Source Documents

- `Improvement Plan.md`
- `1_ANALYZE.md`
- `2_FIND_ISSUES.md`
- `3_EVALUATE.md`
- `4_IMPLEMENT.md`
- `5_PROGRESS.md`

## Program Goals

- Replace the current Flask-first UI with a desktop-native shell and renderer without losing reliable background capture
- Keep the product useful without mandatory AI setup
- Improve search and retrieval quality while preserving local-first privacy guarantees
- Make destructive actions explicit, reversible where possible, and clearly explained
- Ship a product that can run for long periods with stable background behavior

## Cross-Phase Design Principles

- **Local-first:** Keep capture, storage, and baseline browsing functional without cloud dependencies
- **Optional AI:** Treat multimodal provider setup as an enhancement, not a prerequisite
- **Reliable background service:** Optimize for long-running local operation, safe restarts, and visibility into service health
- **Privacy by default:** Protect screenshots, OCR text, embeddings, API keys, and deletion flows
- **Explainable retrieval:** Surface why a result matched, not just that it matched
- **Safe migration:** Preserve access to existing data during the transition away from the current Flask UI

## Deliverable Map

| Phase | Document | Goal | Status | Key Dependency |
|------|----------|------|--------|----------------|
| 01 | `OPENRECALL-DESKTOP-02.md` | Architecture and foundations | Complete | Current-state audit |
| 02 | `OPENRECALL-DESKTOP-03.md` | Desktop shell and core UX | Complete | Phase 01 decisions |
| 03 | `OPENRECALL-DESKTOP-04.md` | AI provider configuration and processing pipeline | Complete | Phase 01 and 02 |
| 04 | `OPENRECALL-DESKTOP-05.md` | Search, timeline, and retrieval rework | Complete | Phase 01 and 03 |
| 05 | `OPENRECALL-DESKTOP-06.md` | Privacy controls, deletion, export, and hardening | Complete | All prior phases |

## Recommended Sequence

1. Finish architecture and migration decisions before UI implementation details
2. Lock the desktop shell, navigation, and settings model before deep provider-specific UX work
3. Define provider abstractions before retrieval and reranking assumptions
4. Finalize retrieval and timeline behaviors before deletion/export semantics that depend on the same data model
5. Close with packaging, hardening, and end-to-end acceptance criteria

## Phase Exit Gates

- **Phase 01 complete when:** service boundaries, persistence strategy, settings domains, migration plan, background behavior, and security baselines are documented
- **Phase 02 complete when:** shell structure, navigation, recorder controls, settings IA, first-run UX, and desktop lifecycle behaviors are specified
- **Phase 03 complete when:** provider modes, optional AI setup, capability boundaries, fallback behavior, and provider health reporting are specified
- **Phase 04 complete when:** hybrid retrieval, timeline UX, explainability, indexing, and long-running performance behavior are specified
- **Phase 05 complete when:** privacy flows, deletion semantics, export behavior, test scope, packaging, and release acceptance are specified

## Open Questions

- Which existing Flask routes, if any, should remain temporarily available during migration and remediation, and how long should that compatibility window last?
- Should provider-backed embeddings become the default active profile in the first desktop release once validated, or remain a secondary profile until retrieval benchmarks are complete?
- Should browser URL hints ship in the first desktop release when they are locally available, or stay deferred until privacy, export, and redaction semantics are fully benchmarked?
- Should retention remain strictly manual in the first desktop release, or is there enough delete-job soak coverage to justify scheduled auto-deletion?

## Status Notes

Use this section to summarize major design decisions as they are approved and to keep the phase status table current.

- 2026-03-15: Phase 01 was completed in `OPENRECALL-DESKTOP-02.md`. The approved foundation keeps capture, OCR, indexing, migration, and retrieval in a Python background service; replaces Flask-rendered UI with Electron main plus React/TypeScript renderer; adopts loopback HTTP plus typed Electron IPC boundaries; and standardizes on a versioned settings file, OS-backed secret storage, and a new SQLite-backed persistence model with legacy import from `recall.db`.
- 2026-03-15: Phase 02 was completed in `OPENRECALL-DESKTOP-03.md`. The approved shell uses Electron main as the sole privileged process, a narrow typed preload bridge, and a React/TypeScript renderer organized around Search, Timeline, Recorder, AI Setup, Settings, and Privacy/Data Management. Recorder controls, tray defaults, launch-on-login behavior, first-run local-first onboarding, and local-only diagnostics are now specified for implementation.
- 2026-03-15: Phase 03 was completed in `OPENRECALL-DESKTOP-04.md`. The approved AI model keeps `Disabled` as the default provider mode, retains built-in local OCR as the baseline text pipeline, adds optional Ollama, LM Studio, and remote OpenAI-compatible integrations through capability-scoped adapters, and requires provider validation, per-capability health reporting, and graceful fallback so capture, timeline, and lexical search continue without provider availability.
- 2026-03-15: Phase 04 was completed in `OPENRECALL-DESKTOP-05.md`. The approved retrieval design adopts a canonical `captures` table plus versioned OCR, embedding, and summary side tables; standardizes on SQLite FTS5 for lexical retrieval; adds filter-first hybrid ranking with optional reranking and explicit explainability labels; and fixes the dedicated Timeline route around day-grouped navigation, surrounding-capture drill-down, and degraded-mode behavior when embeddings or summaries are unavailable.
- 2026-03-15: Phase 05 was completed in `OPENRECALL-DESKTOP-06.md`. The approved privacy and release plan keeps destructive actions and exports as asynchronous tracked jobs, defines explicit delete-all and time-range deletion semantics across screenshots and derived artifacts, makes operations-log export local-only and capture-derived, sets Windows signed packaging as the first supported release target, and requires automated destructive-flow coverage plus a long-running acceptance pass before release.
- 2026-03-15: Program status is now aligned across all redesign phases. The desktop direction is fully defined as a Windows-first Electron plus React shell backed by a Python local service, optional capability-scoped AI integrations, FTS5-plus-embeddings retrieval with explainability, and privacy-by-default deletion and export controls that preserve local-first operation even when provider features are disabled.
