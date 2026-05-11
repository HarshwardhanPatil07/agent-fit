# Implementation Plan: Agent-Fit Assessment Skill

**Branch**: `001-agent-fit` | **Date**: 2026-05-11 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/001-agent-fit/spec.md`

## Summary

Build a Claude Code skill that evaluates any codebase's agent-readiness across 9 technical pillars containing 75+ binary criteria. The overall score is a pass rate percentage (0-100%), and repositories are assigned a maturity level (1-5) via gated 80% progression. Results render as a dark-themed HTML report with per-criterion evidence. The skill is a single markdown command file (`agentfit.md`) that replaces the existing 5-dimension prototype. Zero dependencies — runs entirely within Claude Code.

## Technical Context

**Language/Version**: Markdown (skill definition) + inline HTML/CSS (report template)
**Primary Dependencies**: None — Claude Code only. Optional: `gh` CLI for GitHub API criteria
**Storage**: N/A (no persistent storage; reports generated per run)
**Testing**: Manual validation against 3+ real codebases (Go, Python, TypeScript)
**Target Platform**: Any OS running Claude Code
**Project Type**: Claude Code plugin (skill)
**Performance Goals**: Complete assessment in under 2 minutes for projects up to 10,000 files
**Constraints**: Read-only, no network requests except optional `gh` CLI, zero external deps
**Scale/Scope**: 9 pillars, 75+ criteria, 5 maturity levels, 6 supported languages
**Scoring**: Overall pass rate 0-100% (criteria passed / applicable criteria). Per-pillar scores also 0-100%.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. Agent-First Design | PASS | Structured HTML report, snake_case identifiers, machine-parseable |
| II. Simplicity | PASS | Single command file; no framework, no build step |
| III. Style and Validation | PASS | snake_case criteria, consistent FOUND/MISSING terminology |
| IV. Build System | PASS | Zero dependencies; Claude Code plugin system |
| V. Testing | PASS | Validated against 3+ real codebases before shipping |
| VI. Documentation | PASS | README, command spec, sample report kept consistent |
| VII. Development Environment | PASS | Requires only Claude Code + Git |
| VIII. Debugging and Observability | PASS | Deterministic scores; per-criterion evidence |
| IX. Security and Governance | PASS | Read-only; no secrets; optional `gh` gracefully skipped |
| X. Maintainability | PASS | Single file per concern; replaces prototype in-place |

**Gate result**: PASS — all 10 principles satisfied.

## Project Structure

### Documentation (this feature)

```text
specs/001-agent-fit/
├── plan.md              # This file
├── research.md          # Phase 0: research findings
├── data-model.md        # Phase 1: data model
├── quickstart.md        # Phase 1: quickstart guide
├── contracts/           # Phase 1: output format contract
│   └── report-schema.md
└── tasks.md             # Phase 2: task list (via /speckit-tasks)
```

### Source Code (repository root)

```text
plugins/agentfit/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest (version, description)
├── commands/
│   └── agentfit.md          # The skill definition (single command file)
├── README.md                # Installation and usage docs
└── examples/
    └── sample-report.html   # Example HTML report for reference

.claude-plugin/
└── marketplace.json         # Marketplace manifest
```

**Structure Decision**: Single command file (`agentfit.md`) contains all assessment logic as structured instructions for Claude Code. The HTML report template with dark-theme CSS is embedded inline. This matches the existing plugin structure and the constitution's simplicity principle.

## Complexity Tracking

No violations. Single-file architecture satisfies all constitution principles.
