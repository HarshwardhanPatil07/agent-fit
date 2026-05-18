# Research: Assessment Output Consumability

**Date**: 2026-05-18  
**Feature**: 002-assessment-output-consumability

## R1: JSON Sidecar Scope and Schema Strategy

**Decision**: Output a full assessment JSON sidecar that mirrors the established `AssessmentReport` data model rather than a minimal criteria-only payload.

**Rationale**: The feature goal is consumability across CI artifacts, trend analysis, and dashboards. A complete structured report (project summary, pillars, level progress, highlights, and criterion evidence) removes the need for downstream joins and repeated parsing logic.

**Alternatives considered**:
- Minimal `{criteria}` payload only: insufficient for dashboards and level gating analysis.
- Separate v2 schema file: introduces duplication and drift risk versus existing data model authority.

## R2: Previous-Run Baseline Handling

**Decision**: Use only `/tmp/agentfit-report-{project_name}.json` as the previous-run baseline source.

**Rationale**: This was clarified in spec and keeps behavior deterministic. The command already uses project-specific naming in `/tmp`, so baseline lookup remains simple and compatible with existing flow.

**Alternatives considered**:
- User-configurable baseline path: more flexibility but adds command complexity and ambiguity.
- Multi-run history directory: out of scope for this feature and increases storage lifecycle concerns.

## R3: Metadata Footer Content Source

**Decision**: Render a footer in HTML with assessment date, skill version (`plugin.json`), total evaluated criteria, skipped criteria, assessed repo git SHA, and assessment duration.

**Rationale**: These fields provide traceability and run context needed for auditability and comparison, while all values are already obtainable in current execution context without external dependencies.

**Alternatives considered**:
- Header-only metadata: increases visual clutter in the primary summary area.
- Excluding git SHA/duration: reduces run reproducibility and troubleshooting value.

## R4: Interactive Report Controls

**Decision**: Add inline JavaScript interactions with no external assets for:
- Collapsible pillar sections
- Status filter buttons (`Show All`, `Found`, `Missing`, `Skipped`)
- Criterion name search input
- Maturity level filter

**Rationale**: Inline behavior preserves zero-dependency constraints, works in offline/opened temp files, and materially improves navigation for long criteria lists.

**Alternatives considered**:
- Static report only: does not solve navigability pain for 60+ criteria.
- External JS bundle: violates zero-dependency constraint and adds failure surface.

## R5: Delta Presentation Semantics

**Decision**: Show pass-rate delta when baseline exists (e.g., `67% (+5%)`) and visually highlight criteria whose status changed from prior run.

**Rationale**: Delta-first presentation makes change detection immediate for maintainers and reviewers. Per-criterion highlights provide actionable detail after the top-level trend signal.

**Alternatives considered**:
- Only overall delta, no criterion highlights: too coarse for remediation.
- Full historical table in report: too heavy for current scope.

## R6: Summary Headline Guidance and Narrative Simplification

**Decision**: Expand summary headline examples from 4 to 8-10 and simplify summary template to avoid repeating pass rate.

**Rationale**: More examples reduce stylistic variance in generated narrative, while concise summary text prevents redundancy and improves readability.

**Alternatives considered**:
- Keep existing 4 examples: higher variability in output tone.
- Hard-code one summary sentence: loses contextual flexibility.

## R7: Console Level Gate Status

**Decision**: Extend console summary with a `Levels:` block showing each level percentage, gate threshold, and explicit state (`passed`, `failed gate`, `blocked by previous level`).

**Rationale**: This aligns terminal output with gated maturity logic and helps CI logs communicate why higher levels are unavailable.

**Alternatives considered**:
- Keep pillar-only output: misses the main maturity progression signal.
- Show only final maturity level: hides blocking cause detail.
