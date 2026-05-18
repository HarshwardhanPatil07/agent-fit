# Contract: Report Enhancement Outputs

**Date**: 2026-05-18  
**Feature**: 002-assessment-output-consumability

## Purpose

Define the required output behavior for enhanced Agent-Fit reporting: JSON sidecar parity, metadata footer, interactive controls, delta visibility, summary wording guidance, and console level gate status.

## Output Files

- HTML report: `/tmp/agentfit-report-{project_name}.html`
- JSON sidecar: `/tmp/agentfit-report-{project_name}.json`

Both files MUST be produced on successful assessment completion.

## JSON Sidecar Contract

The sidecar MUST serialize the full assessment report aligned with the `AssessmentReport` model documented in:

- `specs/001-agent-fit/data-model.md`
- `specs/002-assessment-output-consumability/data-model.md` (enhancement additions)

### Required minimum top-level keys

```json
{
  "project_name": "string",
  "language": "string",
  "repo_path": "string",
  "pass_rate": 0,
  "maturity_level": 0,
  "maturity_label": "string",
  "summary_headline": "string",
  "summary_text": "string",
  "metadata": {},
  "delta": {},
  "pillars": [],
  "strengths": [],
  "opportunities": [],
  "level_progress": [],
  "level_gate_status": []
}
```

## HTML Contract

### Footer Metadata

HTML MUST include a footer section showing:

- Assessment date/time
- Skill version (`plugin.json`)
- Total criteria evaluated
- Criteria skipped
- Git SHA of assessed repository
- Assessment duration

### Interactive Behavior

HTML MUST include inline JavaScript (no external assets) supporting:

1. Collapsible pillar sections via pillar header click.
2. Status filter controls: `Show All`, `Found`, `Missing`, `Skipped`.
3. Search input for criterion-name filtering.
4. Level filter to show only criteria from a selected maturity level.
5. Combined filtering logic where visible rows satisfy all active filters.

### Delta Visibility

When baseline sidecar exists at `/tmp/agentfit-report-{project_name}.json`:

- Summary area MUST display pass-rate delta versus previous run.
- Criteria with status changes MUST be visually highlighted.

When baseline is absent or unreadable:

- Report remains valid and complete.
- Delta display is omitted or explicitly marked unavailable.
- No false status-change highlights are shown.

## Summary Text Contract

- Headline examples in command instructions MUST be expanded to 8-10 patterns.
- Summary template MUST not repeat pass rate twice in one narrative block.
- Summary MUST still include maturity level and passed/applicable criteria context.

## Console Output Contract

Existing pillar summary remains, plus a required `Levels:` block:

```text
Levels:
  L1 Functional:   92% ✓ (gate: 80%)
  L2 Documented:   60% ✗ (gate: 80%)
  L3 Standardized: 44% — (blocked by L2)
```

Rules:

- `✓` indicates gate passed for that level.
- `✗` indicates level evaluated but below gate threshold.
- `—` indicates blocked because prerequisite level not passed.

## Error and Compatibility Rules

- Assessment MUST still complete if baseline sidecar is missing.
- Malformed baseline sidecar MUST not abort report generation.
- HTML and JSON outputs MUST remain deterministic for unchanged repository inputs.
