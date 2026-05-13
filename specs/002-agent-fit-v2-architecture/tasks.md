# Tasks: Agent-Fit v2 Architecture & Future

**Input**: Design documents from `specs/002-agent-fit-v2-architecture/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, etc.)
- Include exact file paths in descriptions

---

## Phase 1: Setup

**Purpose**: Version bump and example files that don't depend on skill logic changes

- [x] T001 Bump plugin.json version from 1.0.0 to 2.0.0 in plugins/agentfit/.claude-plugin/plugin.json
- [x] T002 [P] Update marketplace manifest description to reflect v2 capabilities in .claude-plugin/marketplace.json
- [x] T003 [P] Create example .agentfit.yml at plugins/agentfit/examples/agentfit.yml with two custom criteria and two disabled criteria per contracts/agentfit-yml-schema.md
- [x] T004 [P] Create example GitHub Action workflow at plugins/agentfit/examples/github-action.yml showing how to run /agentfit in CI, upload JSON artifact, and post PR comment

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core changes to agentfit.md that ALL user stories depend on

**CRITICAL**: No user story work can begin until this phase is complete

- [x] T005 Add project-type detection logic to Step 1 in plugins/agentfit/commands/agentfit.md — after language detection (line ~18), add instructions to detect project type(s) as one or more of: library, CLI, web_app, api_service, monorepo. Use the detection signals from research.md R1 (manifest files, dependency analysis, directory structure). Store detected types for use by the applicability matrix. Multiple types can be detected simultaneously.
- [x] T006 Add centralized applicability matrix to plugins/agentfit/commands/agentfit.md — after the existing "Criteria Skip Rules" section (line ~30), define the static matrix from data-model.md mapping each project type to criteria to skip. Implement the union resolution rule: skip only if ALL detected types would skip. Replace all existing scattered per-criterion skip rules with references to this matrix.
- [x] T007 Add .agentfit.yml parsing to Step 1 in plugins/agentfit/commands/agentfit.md — after project-type detection, check for `.agentfit.yml` at repo root. If found, parse custom_criteria and disabled_criteria. Validate per contracts/agentfit-yml-schema.md. Warn on invalid YAML or invalid entries. Apply precedence: .agentfit.yml overrides the applicability matrix in both directions.

**Checkpoint**: Project-type detection, applicability matrix, and .agentfit.yml support are wired into Step 1. All downstream user stories can now build on this foundation.

---

## Phase 3: User Story 1 - Project-Type-Aware Assessment (Priority: P1) MVP

**Goal**: Running /agentfit correctly detects project type and skips irrelevant criteria based on the centralized applicability matrix

**Independent Test**: Run /agentfit on a CLI project and verify DAST, health_checks, distributed_tracing, deployment_observability, and product_analytics are skipped with "CLI project" rationale

### Implementation for User Story 1

- [x] T008 [US1] Update Step 2 evaluation instructions in plugins/agentfit/commands/agentfit.md — for each criterion currently with an inline skip rule (deployment_frequency, progressive_rollout, rollback_automation, health_checks, product_analytics, local_services_setup, dast_scanning, distributed_tracing, deployment_observability, heavy_dependency_detection, n_plus_one_detection), remove the inline skip logic and replace with: "Check applicability matrix; if skipped for detected project type(s), mark as skipped with evidence: 'Skipped — {project_type} project'"
- [x] T009 [US1] Add project type badge to the HTML report header in plugins/agentfit/commands/agentfit.md — alongside the existing language badge, display detected project type(s) as badges (e.g., "CLI" "Monorepo"). Use the same badge styling as the language badge.
- [x] T010 [US1] Update the report header summary section in plugins/agentfit/commands/agentfit.md — the narrative summary should mention the detected project type and how many criteria were skipped due to project type

**Checkpoint**: US1 complete — /agentfit detects project type and applies centralized skip rules

---

## Phase 4: User Story 2 - Skill Versioning and Schema Evolution (Priority: P1)

**Goal**: HTML report includes version footer and JSON output includes schema_version

**Independent Test**: Run /agentfit and verify HTML footer shows version, date, SHA and JSON output includes schema_version field

### Implementation for User Story 2

- [x] T011 [US2] Add report metadata footer to the HTML template in plugins/agentfit/commands/agentfit.md — after the ALL CRITERIA section, add a semantic <footer> element displaying: "Agent-Fit v{version}" (from plugin.json), assessment date (ISO 8601), git SHA (from `git rev-parse --short HEAD`), total criteria evaluated, criteria skipped, detected project type(s). Use muted styling consistent with the dark theme.
- [x] T012 [P] [US2] Add schema_version and metadata to JSON output instructions in plugins/agentfit/commands/agentfit.md — extend the JSON sidecar output (written to /tmp/) to include top-level `schema_version` (from plugin.json version), `metadata` object (assessment_date, git_sha, skill_version, project_types, total_criteria, skipped_criteria, custom_criteria_count), and `scores` object (pass_rate, weighted_score, maturity_level) per data-model.md schema
- [x] T013 [US2] Add instructions to read plugin.json version in Step 1 of plugins/agentfit/commands/agentfit.md — at the start of Step 1, read `plugins/agentfit/.claude-plugin/plugin.json` and extract the `version` field for use in the report footer and JSON output

**Checkpoint**: US2 complete — version provenance in both HTML and JSON output

---

## Phase 5: User Story 3 - Custom Criteria Extension (Priority: P2)

**Goal**: .agentfit.yml custom criteria appear in the report and disabled criteria are removed

**Independent Test**: Create .agentfit.yml with one custom criterion and one disabled criterion, run /agentfit, verify the custom criterion appears and the disabled one is absent

### Implementation for User Story 3

- [x] T014 [US3] Add custom criteria integration to Step 2 in plugins/agentfit/commands/agentfit.md — after evaluating all default criteria, if .agentfit.yml defined custom_criteria, evaluate each custom criterion using its `check` and `found_when` fields. Assign to the specified pillar and level. Include in scoring calculations.
- [x] T015 [US3] Add disabled criteria filtering to Step 2 in plugins/agentfit/commands/agentfit.md — before evaluating criteria, remove any criterion whose name appears in .agentfit.yml `disabled_criteria`. These criteria should not appear in the report at all (not found, not missing, not skipped — fully removed from the criteria list and total count).
- [x] T016 [US3] Add custom criteria visual indicator in the HTML report in plugins/agentfit/commands/agentfit.md — custom criteria rows in the ALL CRITERIA table should have a small "Custom" badge next to the criterion name to distinguish them from default criteria

**Checkpoint**: US3 complete — .agentfit.yml fully functional

---

## Phase 6: User Story 4 - Weighted Scoring (Priority: P2)

**Goal**: Report displays weighted score alongside unweighted pass rate

**Independent Test**: Run /agentfit on a project passing high-impact criteria but failing low-impact ones; verify weighted score is higher than unweighted

### Implementation for User Story 4

- [x] T017 [US4] Add impact tier definitions to plugins/agentfit/commands/agentfit.md — after the criteria definitions section, define the three impact tiers (high=3x, medium=2x, low=1x) and assign each criterion to a tier per data-model.md default tier assignments
- [x] T018 [US4] Add weighted score calculation to the scoring methodology section in plugins/agentfit/commands/agentfit.md — compute weighted_score as `sum(weight * passed) / sum(weight * applicable) * 100`. Skipped criteria are excluded from both numerator and denominator. Custom criteria use their `impact_tier` field (default: medium).
- [x] T019 [US4] Add weighted score display to the HTML report header in plugins/agentfit/commands/agentfit.md — next to the existing pass rate percentage, display the weighted score with a label "Weighted: {score}%" and a tooltip/note: "Weighted by criteria impact (high=3x, medium=2x, low=1x)"

**Checkpoint**: US4 complete — weighted scoring visible in report

---

## Phase 7: User Story 5 - CI Integration and GitHub Action (Priority: P3)

**Goal**: Documentation and example workflow for running agent-fit in GitHub Actions

**Independent Test**: Verify the example workflow YAML is valid and the README documents the CI setup process

### Implementation for User Story 5

- [x] T020 [US5] Add CI integration section to plugins/agentfit/README.md — document how to run /agentfit in a GitHub Action, including: prerequisites (Claude Code available in CI), workflow YAML reference (point to examples/github-action.yml), JSON artifact storage for trend tracking, PR comment posting via gh CLI
- [x] T021 [P] [US5] Ensure JSON sidecar is written to a predictable path in plugins/agentfit/commands/agentfit.md — update the JSON output path from `/tmp/agentfit-report-{project_name}.json` to a consistent pattern that CI can reference: `/tmp/agentfit-report-{project_name}-{git_sha}.json`

**Checkpoint**: US5 complete — CI integration documented with working example

---

## Phase 8: User Story 6 - Automated Remediation (Priority: P3)

**Goal**: /agentfit-fix command generates files for missing criteria

**Independent Test**: Run /agentfit, then /agentfit-fix, verify at least one missing criterion now has an appropriate file generated

### Implementation for User Story 6

- [x] T022 [US6] Create the agentfit-fix skill file at plugins/agentfit/commands/agentfit-fix.md — define the skill header, description ("Generates files to fix missing agent-fit criteria"), and invocation instructions. Step 1: locate the most recent JSON sidecar at /tmp/agentfit-report-{project_name}*.json. Error if not found: "No agentfit report found. Run /agentfit first."
- [x] T023 [US6] Write remediation templates in plugins/agentfit/commands/agentfit-fix.md — for each of the 7 priority fix targets (AGENTS.md, .pre-commit-config.yaml, .devcontainer/devcontainer.json, .env.example, dependabot.yml, CODEOWNERS, .github/ISSUE_TEMPLATE/), define a generation template that uses the project language and context from the JSON report. Include --top N parameter support to limit to N highest-priority fixes (prioritized by maturity level, L1 first).
- [x] T024 [US6] Write interactive output mode in plugins/agentfit/commands/agentfit-fix.md — after generating files, present the user with three options: (a) commit changes to current branch with a descriptive message, (b) create a new branch agentfit-fix/{date} and commit there, (c) leave files unstaged for manual review. Wait for user choice before proceeding.

**Checkpoint**: US6 complete — /agentfit-fix generates and stages remediation files

---

## Phase 9: User Story 7 - Validation on Real Repositories (Priority: P1)

**Goal**: End-to-end validation confirms correctness on 5 real repos

**Independent Test**: Run /agentfit on each repo and compare against manual inspection

### Implementation for User Story 7

- [x] T025 [US7] Validate /agentfit on the agent-fit repository itself — run assessment, verify: correct language detection, project-type detection (should be library or CLI), pillar scores consistent with actual repo tooling, report generates without errors, footer shows correct version
- [x] T026 [P] [US7] Validate /agentfit on a public Go repository — clone a Go repo with go.mod, *_test.go, CI workflows. Run assessment, verify: Go-specific criteria correctly evaluated, project-type detection works, no false positives/negatives beyond 5% threshold
- [x] T027 [P] [US7] Validate /agentfit on a public Python repository — clone a Python repo with mypy, pytest, Black. Run assessment, verify: Python-specific criteria (type_check, unit_tests_exist, formatter) correctly evaluated with evidence citing actual config files
- [x] T028 [P] [US7] Validate /agentfit on a public TypeScript repository — clone a TS repo with ESLint, Jest, tsconfig.json. Run assessment, verify: TypeScript-specific criteria correctly evaluated
- [x] T029 [US7] Run quickstart.md validation — follow specs/002-agent-fit-v2-architecture/quickstart.md end-to-end and verify all steps work as documented (project-type detection, .agentfit.yml, weighted scoring, version footer)

**Checkpoint**: US7 complete — all 5 repos validated, false positive/negative rate below 5%

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, examples, and final quality checks

- [x] T030 [P] Update plugins/agentfit/README.md — reflect v2 features: project-type detection, .agentfit.yml support, weighted scoring, version footer, /agentfit-fix command, CI integration
- [x] T031 [P] Update sample report at plugins/agentfit/examples/sample-report.html — reflect v2 report format: project-type badges, weighted score, footer with version/date/SHA, custom criteria badges
- [x] T032 Verify JSON output parses correctly with jq — run /agentfit, pipe JSON sidecar through jq, confirm schema_version, metadata, scores, and criteria fields all present and correctly typed

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001 for version number) — BLOCKS all user stories
- **US1 (Phase 3)**: Depends on Phase 2 (applicability matrix must exist)
- **US2 (Phase 4)**: Depends on Phase 2 (version reading in Step 1)
- **US3 (Phase 5)**: Depends on Phase 2 (.agentfit.yml parsing must exist)
- **US4 (Phase 6)**: Depends on Phase 2 (criteria must be finalized for tier assignment)
- **US5 (Phase 7)**: Depends on US2 (JSON output with schema_version required)
- **US6 (Phase 8)**: Depends on US2 (needs JSON sidecar with stable schema)
- **US7 (Phase 9)**: Depends on US1, US2, US3, US4 (all core features must work before validation)
- **Polish (Phase 10)**: Depends on all user stories

### User Story Dependencies

- **US1 (P1)** + **US2 (P1)** + **US7 (P1)**: Can run in parallel after Phase 2 (US1 and US2 touch different sections of agentfit.md)
- **US3 (P2)** + **US4 (P2)**: Can run in parallel after Phase 2
- **US5 (P3)**: Sequential after US2
- **US6 (P3)**: Sequential after US2
- **US7 (P1)**: Sequential after US1+US2+US3+US4

### Within Each User Story

- Tasks within a story are sequential unless marked [P]
- T008 depends on T005+T006 (can't update criteria until matrix exists)
- T014+T015 depend on T007 (can't integrate custom criteria until parsing exists)

---

## Implementation Strategy

### MVP First (US1 + US2 + Validation)

1. Complete Phase 1: Setup (T001-T004)
2. Complete Phase 2: Foundational (T005-T007)
3. Complete Phase 3: US1 — Project-Type-Aware Assessment
4. Complete Phase 4: US2 — Versioning and Schema
5. Complete Phase 9: US7 — Validation
6. **STOP and VALIDATE**: Core v2 features working and validated

### Incremental Delivery

7. Add US3: Custom Criteria Extension
8. Add US4: Weighted Scoring
9. Add US5: CI Integration Documentation
10. Add US6: Automated Remediation
11. Final Polish (Phase 10)
