# Feature Specification: Assessment Output Consumability

**Feature Branch**: `002-assessment-output-consumability`  
**Created**: 2026-05-18  
**Status**: Draft  
**Input**: User description: "Features that improve the usefulness and consumability of the assessment output."

## Clarifications

### Session 2026-05-18

- Q: Which baseline should trend comparison use for delta calculation? → A: Compare only against the single previous sidecar at `/tmp/agentfit-report-{project_name}.json` if it exists.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Capture Reusable Assessment Data (Priority: P1)

As an engineering lead running assessments in CI, I need each run to produce a machine-readable output file so I can store results as artifacts, compare runs automatically, and feed dashboards.

**Why this priority**: Without machine-readable output, assessment results cannot be reliably reused beyond manual HTML viewing, which blocks automation and trend analysis.

**Independent Test**: Run one assessment and verify a JSON output file is produced in the expected temporary path, includes all required report fields, and can be parsed by an external script.

**Acceptance Scenarios**:

1. **Given** an assessment run completes successfully, **When** output files are generated, **Then** both an HTML report and a JSON report are written for the same project run.
2. **Given** a consumer reads the JSON report, **When** it validates required report fields and criterion details, **Then** the payload matches the documented assessment data model.

---

### User Story 2 - Read and Navigate Large Reports Quickly (Priority: P2)

As a developer reviewing assessment results, I need interactive controls in the HTML report so I can quickly focus on relevant criteria instead of scanning a long static list.

**Why this priority**: The current flat report layout becomes hard to consume at scale, slowing analysis and making actionable items harder to identify.

**Independent Test**: Open a generated HTML report, collapse and expand pillar sections, apply status and level filters, and search by criterion name to verify only matching rows remain visible.

**Acceptance Scenarios**:

1. **Given** the report contains multiple pillars and criteria, **When** the user collapses a pillar, **Then** that pillar's criteria are hidden until expanded again.
2. **Given** mixed criterion statuses and levels, **When** the user applies status, level, or text filters, **Then** only criteria matching all active filters are shown.

---

### User Story 3 - Understand Progress Changes Between Runs (Priority: P3)

As a maintainer tracking repository maturity over time, I need each new report to highlight what changed from the previous run so I can quickly identify improvements and regressions.

**Why this priority**: Teams need immediate visibility into deltas, level gate status, and report context to make release and remediation decisions.

**Independent Test**: Generate two consecutive assessments for the same project with at least one criterion status change and verify the second report shows pass-rate delta, highlights changed criteria, and includes run metadata and level gate status in console output.

**Acceptance Scenarios**:

1. **Given** a previous JSON report exists for the project, **When** a new assessment runs, **Then** the report shows pass-rate delta and visually highlights criteria whose status changed.
2. **Given** summary and console output are generated, **When** users inspect them, **Then** they include non-redundant summary phrasing and clear gated or blocked maturity level status.

---

### Edge Cases

- What happens when no prior JSON report exists for the project? The report shows current metrics without delta values and does not mark criteria as changed.
- How does the system handle a prior JSON report that is unreadable or malformed? The run continues with fresh evaluation and clearly indicates that trend comparison could not be computed.
- What happens when all criteria in a filtered view are excluded? The report keeps the current filter state visible and shows an empty-state message for no matching criteria.
- How does level gating display when a higher level has a high raw percentage but a lower level gate is not met? The output marks the higher level as blocked until prerequisite gates pass.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST generate a machine-readable JSON assessment report for every run in the temporary output location using the same project-specific naming pattern as the HTML output.
- **FR-002**: System MUST include all required assessment report fields and nested entities in the JSON output according to the established report data model used by this project.
- **FR-003**: System MUST include report metadata in the HTML output footer, including assessment date, skill version, total criteria evaluated, skipped criteria count, assessed repository commit identifier, and assessment duration.
- **FR-004**: System MUST provide interactive report controls in HTML without requiring report viewers to install additional tools, including collapsible pillar sections, criterion status filtering, criterion name search, and maturity level filtering.
- **FR-005**: System MUST apply active filters consistently so criteria display reflects the intersection of selected status filters, selected level filter, and text search input.
- **FR-006**: System MUST compare the current assessment with the single previous project JSON sidecar at `/tmp/agentfit-report-{project_name}.json` when available and display a pass-rate delta in the current report.
- **FR-007**: System MUST identify and visually distinguish criteria whose status changed between the current and previous assessment runs.
- **FR-008**: System MUST use an expanded, explicit set of summary headline examples to guide consistent headline generation style across different assessment outcomes.
- **FR-009**: System MUST produce summary text that avoids repeating the same pass-rate value multiple times while preserving total passed versus applicable criteria context.
- **FR-010**: System MUST include maturity level gate status in console output, indicating whether each level is passed, failed its own threshold, or blocked by an unmet prerequisite level.
- **FR-011**: System MUST preserve backward compatibility for report generation when previous-run JSON is missing, including successful report output without comparison data.

### Key Entities *(include if feature involves data)*

- **AssessmentReport**: The complete structured result for a single run, including top-level metrics, maturity outputs, pillar breakdowns, highlights, and level progression.
- **ReportMetadata**: Run-specific context presented to users, including assessment timestamp, skill version, criteria totals, skipped totals, repository commit identifier, and elapsed assessment duration.
- **CriterionDelta**: A per-criterion change descriptor that compares prior and current statuses and flags whether a status transition occurred.
- **FilterState**: The active user-selected controls for report interaction, including status selection, maturity level selection, text query, and pillar collapse state.
- **LevelGateStatus**: The computed gate evaluation per maturity level, including current level percentage, gate threshold, and blocking dependency state from lower levels.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of completed assessments produce both HTML and JSON outputs for the same project run without requiring manual post-processing.
- **SC-002**: In validation runs, at least 95% of JSON outputs are accepted by downstream artifact ingestion and comparison workflows on first attempt.
- **SC-003**: Users can locate a targeted criterion in reports with 60+ rows within 15 seconds using search and filters in usability checks.
- **SC-004**: In comparison-enabled runs, changed criterion statuses and pass-rate deltas are correctly reflected for 100% of intentionally modified test scenarios.
- **SC-005**: In console output verification, 100% of maturity levels show an explicit gate state (passed, failed gate, or blocked), with no ambiguous level status lines.

## Assumptions

- The assessment command continues to write outputs in a temporary filesystem location accessible to CI artifact collection steps.
- The current assessment data model remains the canonical schema source for JSON report structure and field naming.
- Skill version metadata is available from existing plugin metadata and can be included in report output without changing the user invocation flow.
- Interactive report features are expected to run in modern browsers with inline scripting enabled.
- The most recent baseline for trend comparison is the existing JSON sidecar at `/tmp/agentfit-report-{project_name}.json`; if absent, the run proceeds without delta output.
