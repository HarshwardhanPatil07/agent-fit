# Data Model: Assessment Output Consumability

**Date**: 2026-05-18  
**Feature**: 002-assessment-output-consumability

## Entities

### AssessmentReport

The top-level report object remains the canonical machine-readable artifact and is written to `/tmp/agentfit-report-{project_name}.json`.

| Field | Type | Description |
|-------|------|-------------|
| project_name | string | Repository/project name |
| language | string | Primary language (or composite badge label) |
| repo_path | string | Repository path or local directory path |
| pass_rate | number | Overall pass rate percentage |
| maturity_level | number | Highest unlocked level (1-5) |
| maturity_label | string | Human label for maturity level |
| summary_headline | string | Narrative headline |
| summary_text | string | 1-2 sentence summary without duplicated pass-rate phrase |
| metadata | ReportMetadata | Run metadata for traceability |
| delta | ReportDelta | Previous-run comparison outputs |
| pillars | Pillar[] | Per-pillar results and criteria |
| strengths | Highlight[] | Top strengths |
| opportunities | Highlight[] | Top opportunities |
| level_progress | LevelProgress[] | L1-L5 percentages |
| level_gate_status | LevelGateStatus[] | Gate/unlock status by level |

### ReportMetadata

Run context rendered in the HTML footer and available in JSON.

| Field | Type | Description |
|-------|------|-------------|
| assessed_at | string (ISO 8601) | Assessment timestamp |
| skill_version | string | Version from `plugins/agentfit/.claude-plugin/plugin.json` |
| criteria_evaluated | number | Count of applicable criteria (`found` + `missing`) |
| criteria_skipped | number | Count of skipped criteria |
| git_sha | string | Commit SHA of assessed repository (short or full) |
| assessment_duration_seconds | number | Total elapsed runtime in seconds |

### ReportDelta

Top-level comparison summary against previous sidecar baseline.

| Field | Type | Description |
|-------|------|-------------|
| baseline_found | boolean | Whether baseline sidecar existed and parsed |
| baseline_path | string | Expected baseline path (`/tmp/agentfit-report-{project_name}.json`) |
| pass_rate_previous | number \| null | Prior pass rate when baseline exists |
| pass_rate_current | number | Current pass rate |
| pass_rate_delta | number \| null | `current - previous` |
| changed_criteria_count | number | Number of criteria whose status changed |

### Pillar

| Field | Type | Description |
|-------|------|-------------|
| name | string | Pillar display name |
| criteria_passed | number | Criteria marked `found` |
| criteria_total | number | Applicable criteria count |
| percentage | number | Pillar pass percentage |
| criteria | Criterion[] | Criterion rows in pillar |
| collapsed_default | boolean | Whether pillar is collapsed on initial render |

### Criterion

| Field | Type | Description |
|-------|------|-------------|
| name | string | Criterion identifier |
| status | enum | `found`, `missing`, `skipped` |
| score | string | `1/1`, `0/1`, or `—/—` |
| evidence | string | Supporting evidence text |
| level | number | Maturity level of criterion |
| status_changed | boolean | Whether status differs from baseline |
| previous_status | enum \| null | Previous baseline status if available |

### FilterState

UI state tracked by inline report interactions.

| Field | Type | Description |
|-------|------|-------------|
| status_filter | enum | `all`, `found`, `missing`, `skipped` |
| level_filter | number \| `all` | Selected maturity level filter |
| search_query | string | Case-insensitive criterion name query |

### LevelGateStatus

Console and JSON representation of level gate decisions.

| Field | Type | Description |
|-------|------|-------------|
| level | number | 1-5 |
| label | string | Level label (Functional, Documented, etc.) |
| percentage | number | Current completion percentage |
| gate_threshold | number | Gate threshold (80) |
| state | enum | `passed`, `failed_gate`, `blocked` |
| blocked_by_level | number \| null | Prior level causing block, when state is `blocked` |

## Relationships

```text
AssessmentReport 1───1 ReportMetadata
AssessmentReport 1───1 ReportDelta
AssessmentReport 1───* Pillar 1───* Criterion
AssessmentReport 1───* LevelProgress
AssessmentReport 1───* LevelGateStatus
AssessmentReport 1───* Highlight
```

## Validation Rules

- JSON sidecar MUST include required `AssessmentReport` fields and nested objects.
- `criteria_evaluated + criteria_skipped` MUST equal total criteria considered in scoring pipeline.
- `pass_rate_delta` MUST be null when `baseline_found` is false.
- `status_changed` MUST be true only when both current and previous statuses exist and differ.
- `level_gate_status.state` MUST follow gated progression rules:
  - `passed` when current level >= 80% and all previous levels passed
  - `failed_gate` when current level < 80% and all previous levels passed
  - `blocked` when any previous level failed

## State Transitions

No persistent server-side state transitions. Each run computes a fresh `AssessmentReport`; optional delta fields are derived from baseline sidecar comparison.
