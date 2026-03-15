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
- [ ] **Run `OPENRECALL-DESKTOP-06.md`:** Complete the Privacy Controls, Deletion, Export, and Hardening document after prior phase decisions are available.
- [ ] **Update program status in `OPENRECALL-DESKTOP-01.md`:** After the phase files are completed, update the deliverable map statuses, summarize major approved decisions, and record any remaining open questions.

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
| 04 | `OPENRECALL-DESKTOP-05.md` | Search, timeline, and retrieval rework | Not Started | Phase 01 and 03 |
| 05 | `OPENRECALL-DESKTOP-06.md` | Privacy controls, deletion, export, and hardening | Not Started | All prior phases |

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

- Which existing Flask routes will remain temporarily available during migration?
- What minimum viable desktop packaging target comes first?
- How much existing OCR and indexing logic can be reused without changing storage contracts?
- Which provider capabilities are mandatory for the first desktop release?

## Status Notes

Use this section to summarize major design decisions as they are approved and to keep the phase status table current.

- 2026-03-15: Phase 01 was completed in `OPENRECALL-DESKTOP-02.md`. The approved foundation keeps capture, OCR, indexing, migration, and retrieval in a Python background service; replaces Flask-rendered UI with Electron main plus React/TypeScript renderer; adopts loopback HTTP plus typed Electron IPC boundaries; and standardizes on a versioned settings file, OS-backed secret storage, and a new SQLite-backed persistence model with legacy import from `recall.db`.
- 2026-03-15: Phase 02 was completed in `OPENRECALL-DESKTOP-03.md`. The approved shell uses Electron main as the sole privileged process, a narrow typed preload bridge, and a React/TypeScript renderer organized around Search, Timeline, Recorder, AI Setup, Settings, and Privacy/Data Management. Recorder controls, tray defaults, launch-on-login behavior, first-run local-first onboarding, and local-only diagnostics are now specified for implementation.
- 2026-03-15: Phase 03 was completed in `OPENRECALL-DESKTOP-04.md`. The approved AI model keeps `Disabled` as the default provider mode, retains built-in local OCR as the baseline text pipeline, adds optional Ollama, LM Studio, and remote OpenAI-compatible integrations through capability-scoped adapters, and requires provider validation, per-capability health reporting, and graceful fallback so capture, timeline, and lexical search continue without provider availability.
