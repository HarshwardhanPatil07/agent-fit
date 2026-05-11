<!-- Sync Impact Report
Version change: 0.0.0 → 1.0.0 (initial ratification)
Modified principles: N/A (initial)
Added sections:
  - 10 Core Principles (I–X)
  - Development Workflow
  - Quality Gates
  - Governance
Removed sections: N/A
Templates requiring updates:
  - .specify/templates/plan-template.md ✅ (Constitution Check section is dynamic)
  - .specify/templates/spec-template.md ✅ (no changes needed)
  - .specify/templates/tasks-template.md ✅ (no changes needed)
Follow-up TODOs: None
-->

# Agent-Fit Constitution

## Core Principles

### I. Agent-First Design

Every feature, interface, and architectural decision MUST prioritize
machine consumability over human convenience. Software produced by
this project MUST be usable by AI agents without human intervention.
All assessments, scoring rubrics, and recommendations MUST reflect
the agent-first worldview: APIs before UIs, structured output before
prose, programmatic access before manual workflows.

### II. Simplicity

Start simple. Follow YAGNI — do not build for hypothetical future
requirements. Prefer fewer files, fewer abstractions, and fewer
moving parts. A flat structure is better than a deep one. A single
well-written file is better than a framework. Complexity MUST be
justified in writing before it is introduced.

### III. Style and Validation

All code and content MUST follow consistent formatting and linting
rules. Use project-level linters and formatters where available.
Naming conventions MUST be lowercase with hyphens for branches and
files. Commit messages MUST use the `type: description` format
(e.g., `docs:`, `skill:`, `command:`). Terminology MUST stay
consistent across all artifacts (`FOUND`, `PARTIAL`, `MISSING`,
score rubric).

### IV. Build System

The project MUST remain zero-dependency for end users. The agentfit
skill requires only Claude Code — no external packages, no API keys,
no installs. Build and distribution MUST use the Claude Code plugin
system (`.claude-plugin/` manifests). Version bumps MUST update
`plugin.json` and follow semantic versioning.

### V. Testing

All assessment logic MUST be validated against real codebases before
shipping. Test on at least 3 different project types to confirm
findings are useful and non-obvious. Static analysis checks MUST NOT
make network requests or modify the target codebase. When tests are
included in feature specs, TDD (Red-Green-Refactor) MUST be followed.

### VI. Documentation

Documentation MUST serve two audiences: human developers and AI
agents. Every skill, command, and scoring dimension MUST have clear,
structured documentation. README, CONTRIBUTING, and vision docs MUST
stay consistent with each other. No contradiction between SKILL.md,
command specs, and sample reports. Changes to output format MUST
include before/after snippets.

### VII. Development Environment

Development requires only Claude Code and a Git repository. No
additional tooling, CI pipelines, or infrastructure is required in
the current phase. Contributors MUST follow the branch naming
convention `<name>/<short-purpose>` with lowercase hyphens. PRs
MUST be focused (one change per PR) and reviewed by at least one
teammate before merge.

### VIII. Debugging and Observability

Assessment output MUST be structured and machine-parseable. Scores
MUST be deterministic given the same codebase state — identical
inputs produce identical reports. Findings MUST include enough
context for a developer to locate and fix the gap without additional
investigation. Error states MUST be surfaced clearly, not swallowed.

### IX. Security and Governance

The skill MUST be read-only — it MUST NOT modify the target codebase
or make network requests. No secrets, credentials, or sensitive data
may be committed to the repository. The `.claude-plugin/plugin.json`
manifest fields `name`, `owner`, and `source` MUST NOT be changed
without team discussion. All PRs MUST pass the content quality
checklist before merge.

### X. Maintainability

Keep the repository structure clean. Do not add unnecessary files or
folders. Prefer editing existing files over creating new ones. Every
file in the repo MUST have a clear purpose. Dead code and unused
artifacts MUST be removed promptly. Changes MUST be small and
focused — large refactors require explicit team agreement.

## Development Workflow

1. Branch from `main` using `<name>/<short-purpose>` convention.
2. Make focused, small changes aligned with a single purpose.
3. Write clear commit messages: `type: description`.
4. Open a PR with a short summary of what changed and why.
5. Request at least one teammate review.
6. Merge only after review approval and quality checklist pass.

## Quality Gates

All PRs MUST satisfy before merge:

- Wording is clear and consistent with the project vision.
- No contradiction between skill definition, command spec, and
  sample report.
- Terminology is consistent across all artifacts.
- No unnecessary files or folders added.
- If output format changed, before/after snippets included.
- Branch naming convention followed.
- Commit message format followed.

## Governance

This constitution supersedes all other development practices for the
Agent-Fit project. Amendments require:

1. A written proposal describing the change and rationale.
2. Review and approval by at least one core team member.
3. Version bump following semantic versioning:
   - MAJOR: Principle removals or incompatible redefinitions.
   - MINOR: New principles or materially expanded guidance.
   - PATCH: Clarifications, wording, or typo fixes.
4. Update to `LAST_AMENDED_DATE` upon any change.
5. Consistency propagation check across all dependent templates.

All PRs and reviews MUST verify compliance with this constitution.

**Version**: 1.0.0 | **Ratified**: 2026-05-11 | **Last Amended**: 2026-05-11
