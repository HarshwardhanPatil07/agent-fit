# Implementation Plan: Agent-Fit v2 Architecture & Future

**Branch**: `shreyash/002-agent-fit-v2-architecture` | **Date**: 2026-05-13 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/002-agent-fit-v2-architecture/spec.md`

## Summary

Extend the agent-fit skill with v2 architectural improvements: centralized project-type detection with an applicability matrix for skip rules, custom criteria extension via `.agentfit.yml`, weighted scoring as a secondary signal, skill versioning in reports and JSON output, a separate `/agentfit-fix` remediation command, CI integration documentation with GitHub Action examples, and end-to-end validation against 5 real repositories. The core skill file (`agentfit.md`) is extended in-place; `/agentfit-fix` is a new separate skill file.

## Technical Context

**Language/Version**: Markdown (skill definition) + inline HTML/CSS/JS (report template)
**Primary Dependencies**: None — Claude Code only. Optional: `gh` CLI for GitHub API criteria
**Storage**: N/A (reports generated per run; JSON sidecar written to `/tmp/`)
**Testing**: Manual validation against 5 real codebases (agent-fit, Go, Python, TypeScript, quickstart)
**Target Platform**: Any OS running Claude Code
**Project Type**: Claude Code plugin (skill)
**Performance Goals**: Complete assessment in under 2 minutes for projects up to 10,000 files
**Constraints**: Read-only for `/agentfit` (existing constraint). `/agentfit-fix` is write-mode (separate skill). Zero external dependencies. No network requests except optional `gh` CLI.
**Scale/Scope**: 9 pillars, 75+ criteria + custom criteria, 5 maturity levels, 5 project types, weighted scoring with 3 impact tiers

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. Agent-First Design | PASS | JSON output with `schema_version` enables machine consumption; `.agentfit.yml` is machine-readable config; weighted scoring adds structured signal |
| II. Simplicity | PASS | Extends existing single file (`agentfit.md`); `/agentfit-fix` is one additional file; no frameworks or build steps added |
| III. Style and Validation | PASS | snake_case for criteria names, project types, and YAML keys; consistent FOUND/MISSING/SKIPPED terminology preserved |
| IV. Build System | PASS | Zero dependencies maintained; plugin.json version bumped per semver; no external packages introduced |
| V. Testing | PASS | Validated against 5 real codebases before shipping; validation tasks T024-T028 completed |
| VI. Documentation | PASS | CI integration documented with copy-paste workflow YAML; `.agentfit.yml` schema documented; report footer adds version provenance |
| VII. Development Environment | PASS | Requires only Claude Code + Git; no additional tooling |
| VIII. Debugging and Observability | PASS | schema_version enables output format tracking; report footer adds assessment metadata; weighted score is transparent (tier assignments visible) |
| IX. Security and Governance | PASS | `/agentfit` remains read-only; `/agentfit-fix` is separate skill with write permissions; no secrets handled; `.agentfit.yml` validated before use |
| X. Maintainability | PASS | Centralized applicability matrix replaces scattered skip rules (net reduction in maintenance surface); version bumping codified in process |

**Gate result**: PASS — all 10 principles satisfied.

## Project Structure

### Documentation (this feature)

```text
specs/002-agent-fit-v2-architecture/
├── plan.md              # This file
├── research.md          # Phase 0: research findings
├── data-model.md        # Phase 1: data model
├── quickstart.md        # Phase 1: quickstart guide
├── contracts/           # Phase 1: output format contracts
│   └── agentfit-yml-schema.md
└── tasks.md             # Phase 2: task list (via /speckit-tasks)
```

### Source Code (repository root)

```text
plugins/agentfit/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest — version bumped to 2.0.0
├── commands/
│   ├── agentfit.md              # Extended: project-type detection, applicability matrix,
│   │                            #   .agentfit.yml support, weighted scoring, version footer
│   └── agentfit-fix.md          # NEW: remediation command (separate skill file)
├── README.md                    # Updated: v2 features, .agentfit.yml docs, CI docs
└── examples/
    ├── sample-report.html       # Updated: reflects v2 report format (footer, weighted score)
    ├── agentfit.yml              # NEW: example .agentfit.yml configuration
    └── github-action.yml        # NEW: example GitHub Action workflow

.claude-plugin/
└── marketplace.json             # Updated: description reflects v2 capabilities
```

**Structure Decision**: Extends the existing single-file architecture. The only new skill file is `agentfit-fix.md` (separate from `agentfit.md` because it has write permissions, per Constitution IX). All other changes are modifications to existing files. Example files added for `.agentfit.yml` and GitHub Action workflow.

## Complexity Tracking

No violations. The addition of `agentfit-fix.md` as a second skill file is justified by the separation of read-only assessment from write-mode remediation (Constitution IX: Security and Governance).
