# Tasks: Assessment Output Consumability

**Input**: Design documents from `/specs/002-assessment-output-consumability/`  
**Prerequisites**: `plan.md` (required), `spec.md` (required), `research.md`, `data-model.md`, `contracts/report-enhancements.md`, `quickstart.md`

**Tests**: No TDD-only test-writing requirement was explicitly requested; tasks include implementation plus manual validation checkpoints derived from independent test criteria.

**Organization**: Tasks are grouped by user story so each story can be implemented and validated independently.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no direct dependency)
- **[Story]**: User story label (`[US1]`, `[US2]`, `[US3]`)
- Every task includes an exact file path

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Prepare command-file editing scope and validation references.

- [X] T001 Review current report flow anchors in `plugins/agentfit/commands/agentfit.md` (Step 1, Step 3 summary, Step 4 HTML/JSON output, Step 5 console output)
- [X] T002 Confirm skill version source and expected value in `plugins/agentfit/.claude-plugin/plugin.json` for footer metadata wiring
- [X] T003 Capture implementation checklist comments in `specs/002-assessment-output-consumability/quickstart.md` for run validation order

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Establish shared report data shape and baseline behavior used by all stories.

**⚠️ CRITICAL**: No user story work should start until this phase is complete.

- [X] T004 Define canonical report object fields and naming alignment in `plugins/agentfit/commands/agentfit.md` to match `AssessmentReport` contract
- [X] T005 Add baseline sidecar read/error-handling instruction block in `plugins/agentfit/commands/agentfit.md` for `/tmp/agentfit-report-{project_name}.json`
- [X] T006 Specify deterministic fallback behavior when baseline JSON is missing or malformed in `plugins/agentfit/commands/agentfit.md`

**Checkpoint**: Shared report object and baseline semantics are defined and ready for story-level implementation.

---

## Phase 3: User Story 1 - Capture Reusable Assessment Data (Priority: P1) 🎯 MVP

**Goal**: Always produce a machine-readable JSON sidecar that is schema-aligned and usable by CI/dashboards.

**Independent Test**: Run `/agentfit` once and confirm both HTML and JSON outputs are produced, with JSON parseable and populated with required top-level fields.

### Implementation for User Story 1

- [X] T007 [US1] Update Step 4 output instruction in `plugins/agentfit/commands/agentfit.md` to require writing `/tmp/agentfit-report-{project_name}.json` on every successful run
- [X] T008 [US1] Expand JSON sidecar contract details in `plugins/agentfit/commands/agentfit.md` to include full report payload (metadata, pillars, strengths/opportunities, level progress, gate status)
- [X] T009 [US1] Require synchronized HTML+JSON generation success criteria in `plugins/agentfit/commands/agentfit.md` (same run, same project naming)
- [X] T010 [US1] Update command summary block in `plugins/agentfit/commands/agentfit.md` to print both HTML and JSON output paths
- [X] T011 [P] [US1] Update output expectations in `plugins/agentfit/README.md` to document JSON sidecar usage for artifacts/comparisons

**Checkpoint**: User Story 1 is independently verifiable via one-run dual-output generation.

---

## Phase 4: User Story 2 - Read and Navigate Large Reports Quickly (Priority: P2)

**Goal**: Make HTML report navigable with collapsible sections and combined filters.

**Independent Test**: Open generated HTML and verify collapse/expand, status filter, search filter, and level filter all work together for criteria visibility.

### Implementation for User Story 2

- [X] T012 [US2] Add filter toolbar and level selector markup in HTML template section of `plugins/agentfit/commands/agentfit.md`
- [X] T013 [US2] Add style rules for collapsed pillars, active filter controls, and changed-row emphasis in `plugins/agentfit/commands/agentfit.md`
- [X] T014 [US2] Add inline JavaScript for collapsible pillar toggling in `plugins/agentfit/commands/agentfit.md`
- [X] T015 [US2] Add inline JavaScript for status filtering and criterion-name search in `plugins/agentfit/commands/agentfit.md`
- [X] T016 [US2] Add inline JavaScript for maturity-level filtering with combined filter logic in `plugins/agentfit/commands/agentfit.md`
- [X] T017 [US2] Add empty-result handling behavior and message requirements in `plugins/agentfit/commands/agentfit.md` when filters return zero rows
- [X] T018 [P] [US2] Refresh visual example in `plugins/agentfit/examples/sample-report.html` to reflect interactive controls

**Checkpoint**: User Story 2 interactions work without external dependencies and can be validated in-browser.

---

## Phase 5: User Story 3 - Understand Progress Changes Between Runs (Priority: P3)

**Goal**: Surface trend deltas, report metadata, improved summary guidance, and explicit level gate statuses.

**Independent Test**: Run assessment twice with at least one criterion status change; verify delta output, changed-row highlighting, footer metadata, simplified summary text, and level gate lines in console output.

### Implementation for User Story 3

- [X] T019 [US3] Add report footer requirements (assessment date, skill version, counts, git SHA, duration) to HTML template instructions in `plugins/agentfit/commands/agentfit.md`
- [X] T020 [US3] Add pass-rate delta presentation and changed-criterion highlight rules in `plugins/agentfit/commands/agentfit.md` based on baseline comparison
- [X] T021 [US3] Expand summary headline examples to 8-10 patterns in `plugins/agentfit/commands/agentfit.md`
- [X] T022 [US3] Simplify summary narrative template in `plugins/agentfit/commands/agentfit.md` to avoid duplicate pass-rate wording
- [X] T023 [US3] Extend Step 5 console output in `plugins/agentfit/commands/agentfit.md` with `Levels:` gate status lines (`✓`, `✗`, `— blocked`)
- [X] T024 [P] [US3] Document trend/delta and level-gate console behavior in `plugins/agentfit/README.md`

**Checkpoint**: User Story 3 clearly communicates change-over-time and gate-state context in HTML + console outputs.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Final consistency and end-to-end validation across stories.

- [X] T025 [P] Align wording across `plugins/agentfit/commands/agentfit.md` and `plugins/agentfit/README.md` for `found`/`missing`/`skipped` and maturity gate terms
- [X] T026 Validate all quickstart scenarios in `specs/002-assessment-output-consumability/quickstart.md` and record any corrections back into `plugins/agentfit/commands/agentfit.md`
- [X] T027 [P] Verify JSON sidecar contract consistency against `specs/001-agent-fit/data-model.md` and `specs/002-assessment-output-consumability/data-model.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: Starts immediately.
- **Phase 2 (Foundational)**: Depends on Phase 1; blocks all user stories.
- **Phase 3 (US1)**: Depends on Phase 2; defines MVP.
- **Phase 4 (US2)**: Depends on Phase 2; can proceed after MVP or in parallel staffing model.
- **Phase 5 (US3)**: Depends on Phase 2 and should follow US1 baseline semantics.
- **Phase 6 (Polish)**: Depends on completion of all selected user stories.

### User Story Dependencies

- **US1 (P1)**: No dependency on other user stories; first deliverable (MVP).
- **US2 (P2)**: Independent of US1 logic but shares same command file; schedule after US1 for lower merge risk.
- **US3 (P3)**: Depends on baseline/JSON semantics from foundational + US1 outputs.

### Within Each User Story

- Contract/instruction updates before output-format refinements.
- Core command edits before documentation/example refreshes.
- Story checkpoint validation before moving to the next priority.

### Parallel Opportunities

- `T011` can run in parallel with late US1 command refinements once JSON contract is stable.
- `T018` can run in parallel with final US2 command polishing.
- `T024`, `T025`, and `T027` can run in parallel near completion since they touch separate files.

---

## Parallel Example: User Story 2

```bash
# Parallelizable after core US2 command logic is in place:
Task: "Refresh visual example in plugins/agentfit/examples/sample-report.html"
Task: "Add empty-result handling behavior and message requirements in plugins/agentfit/commands/agentfit.md"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 + Phase 2.
2. Complete Phase 3 (US1).
3. Validate dual-output behavior (HTML + JSON sidecar).
4. Stop for MVP review and CI artifact integration check.

### Incremental Delivery

1. Deliver US1 for machine-readable output.
2. Add US2 interactive report consumption improvements.
3. Add US3 trend/delta + metadata + gate-status messaging.
4. Finish with cross-cutting contract and wording validation.

### Parallel Team Strategy

1. One contributor handles core command-file logic in `plugins/agentfit/commands/agentfit.md`.
2. Another contributor updates `plugins/agentfit/README.md` and `plugins/agentfit/examples/sample-report.html` once each story contract stabilizes.
3. Final pass validates quickstart and schema consistency.

---

## Notes

- `[P]` tasks are selected only when they touch separate files and do not require unfinished upstream tasks.
- User story labels map directly to `spec.md` priorities for traceability.
- Keep scope focused on report consumability improvements; avoid unrelated criteria or scoring-rule changes.
