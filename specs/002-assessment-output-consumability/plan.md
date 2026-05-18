# Implementation Plan: Assessment Output Consumability

**Branch**: `002-assessment-output-consumability` | **Date**: 2026-05-18 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `specs/002-assessment-output-consumability/spec.md`

## Summary

Improve the Agent-Fit report experience by producing a full JSON sidecar alongside HTML, adding report metadata, enabling interactive criteria navigation, surfacing deltas from the prior run baseline, expanding summary headline guidance, and adding explicit maturity gate status lines to console output. The implementation remains a single command file update in `plugins/agentfit/commands/agentfit.md` with no end-user dependency changes.

## Technical Context

**Language/Version**: Markdown command spec with inline HTML/CSS/JavaScript template  
**Primary Dependencies**: None for end users; existing shell tools (`git`) and optional `gh` usage remain unchanged  
**Storage**: Temporary output files in `/tmp` (`.html` and `.json` report pair)  
**Testing**: Manual command runs on representative repositories + output contract validation against the existing data model  
**Target Platform**: Claude/Cursor agent environments on macOS/Linux with browser opener fallback behavior  
**Project Type**: Claude Code plugin command enhancement  
**Performance Goals**: Maintain end-to-end assessment runtime under 2 minutes for repositories up to 10,000 files; keep report interactions responsive for 60+ criteria rows  
**Constraints**: Read-only assessment of target repository; zero external runtime dependencies in generated report; preserve current report generation flow and compatibility when no prior JSON baseline exists  
**Scale/Scope**: One command file updated, with output contract and planning artifacts for JSON schema alignment, interactive UI behavior, and trend comparison rules

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. Agent-First Design | PASS | JSON sidecar and delta metadata increase machine consumability and automation readiness |
| II. Simplicity | PASS | Change remains in existing command file and output contract; no framework or service additions |
| III. Style and Validation | PASS | Existing terminology (`found`/`missing`/`skipped`) and report schema consistency are preserved |
| IV. Build System | PASS | Zero-dependency user experience remains unchanged |
| V. Testing | PASS | Plan includes validation across real repos and schema/output checks |
| VI. Documentation | PASS | Contract and quickstart updates document new outputs and console behavior |
| VII. Development Environment | PASS | No new toolchain requirements for contributors |
| VIII. Debugging and Observability | PASS | Footer metadata and level gate status improve traceability and diagnostics |
| IX. Security and Governance | PASS | Assessment remains read-only and does not expand network behavior |
| X. Maintainability | PASS | Focused scope with minimal file surface and explicit contracts |

**Gate result (pre-design)**: PASS

## Project Structure

### Documentation (this feature)

```text
specs/002-assessment-output-consumability/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── report-enhancements.md
└── tasks.md
```

### Source Code (repository root)

```text
plugins/agentfit/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── agentfit.md
├── README.md
└── examples/
    └── sample-report.html
```

**Structure Decision**: Keep the single-file command architecture and update `plugins/agentfit/commands/agentfit.md` as the authoritative logic/template source. Add only spec-side design artifacts under `specs/002-assessment-output-consumability/` to define behavior and contracts before task breakdown.

## Phase 0: Research Summary

Phase 0 resolves key decisions for sidecar schema shape, metadata derivation, trend baseline semantics, interactive control behavior, and summary/gate output phrasing. See `research.md`.

## Phase 1: Design Outputs

- Data model updates for report metadata, criterion deltas, filter state, and level gate lines are captured in `data-model.md`.
- Output and behavior contract is captured in `contracts/report-enhancements.md`.
- Validation flow and manual run steps are captured in `quickstart.md`.
- Agent context update script executed for `cursor-agent`.

## Post-Design Constitution Check

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. Agent-First Design | PASS | JSON sidecar + explicit deltas and metadata make outputs automation-ready |
| II. Simplicity | PASS | No architectural sprawl; enhancements layered on existing template |
| III. Style and Validation | PASS | Consistent scoring/status vocabulary and deterministic baseline rules |
| IV. Build System | PASS | No new dependencies or build steps introduced |
| V. Testing | PASS | Quickstart includes reproducible validation checks and baseline/no-baseline cases |
| VI. Documentation | PASS | Plan artifacts define schema, contract, and runbook updates clearly |
| VII. Development Environment | PASS | Contributors still need only existing repo tooling |
| VIII. Debugging and Observability | PASS | Footer, duration, SHA, and gate lines improve troubleshooting context |
| IX. Security and Governance | PASS | Read-only analysis scope retained |
| X. Maintainability | PASS | Focused file changes and explicit contract boundaries reduce drift risk |

**Gate result (post-design)**: PASS

## Complexity Tracking

No constitution violations or complexity exemptions required.
