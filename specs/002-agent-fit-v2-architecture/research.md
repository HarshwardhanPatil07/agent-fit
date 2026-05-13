# Research: Agent-Fit v2 Architecture & Future

**Date**: 2026-05-13
**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

## R1: Project-Type Detection Signals

**Decision**: Detect project type from manifest files, dependency presence, and directory structure — no build system execution required.

**Rationale**: Build system execution would violate the read-only constraint and introduce timing/environment dependencies. Manifest-based detection is deterministic and fast.

**Detection signals per type**:

| Type | Primary Signals | Secondary Signals |
|------|----------------|-------------------|
| Library | `package.json` with no `bin`/no server framework; `setup.py`/`pyproject.toml` with no CLI entry_points; pure module exports | No `Dockerfile`, no deployment configs, no HTTP framework imports |
| CLI | `package.json` with `bin` field; `pyproject.toml` with `[project.scripts]`; `main.go` with `cobra`/`urfave/cli`/`flag` imports; Cargo.toml with `[[bin]]` | No HTTP server imports, no web framework deps |
| Web App | HTTP framework in deps (`react`, `next`, `vue`, `angular`, `flask`, `django`, `rails`, `express` + template/static dirs) | `public/`, `static/`, `templates/` directories; HTML/CSS/JS source files |
| API Service | HTTP framework in deps (`fastapi`, `gin`, `echo`, `express`, `actix-web`) WITHOUT frontend assets | OpenAPI/Swagger specs, `.proto` files, no `public/`/`static/` dirs |
| Monorepo | `pnpm-workspace.yaml`, `lerna.json`, `nx.json`, `turbo.json`, Cargo workspace `[workspace]`, multiple `go.mod` files | `packages/`, `apps/`, `services/` directory structure |

**Alternatives considered**: Running `npm ls` or `pip list` — rejected because requires installed dependencies and introduces environment dependency.

## R2: Multi-Type Resolution Strategy

**Decision**: Union approach — detect ALL matching types, skip a criterion only if ALL types would skip it.

**Rationale**: The most permissive approach prevents false skips. A monorepo containing a CLI and a web app should evaluate web app criteria (like product_analytics) even though CLI alone would skip them.

**Alternatives considered**:
- Primary type wins (first match): Too simplistic, loses information from secondary types.
- Most permissive type: Equivalent to union in practice but harder to reason about.
- User-declared type: Adds friction; detection should work automatically.

## R3: `.agentfit.yml` Schema Design

**Decision**: Simple YAML schema with two top-level keys: `custom_criteria` and `disabled_criteria`. User config always overrides project-type matrix.

**Rationale**: Minimal schema keeps parsing within skill instructions feasible (no external validator). Two keys cover the two use cases: adding project-specific checks and removing irrelevant defaults.

**Schema**:
```yaml
custom_criteria:
  - name: "criterion_snake_case_name"
    pillar: "Pillar Name"     # Must match one of the 9 pillar names
    level: "L2"               # L1-L5
    check: "What to look for"
    found_when: "Condition for FOUND status"

disabled_criteria:
  - "criterion_name_to_remove"
```

**Alternatives considered**: JSON schema with `$schema` reference — rejected per Constitution II (simplicity). Full JSONSchema validation — overkill for a YAML file with 2 keys.

## R4: Weighted Scoring Model

**Decision**: Three impact tiers (high=3x, medium=2x, low=1x) displayed alongside unweighted pass rate. Default tiers assigned per criterion. Custom weights not supported in v2.

**Rationale**: The v1 research doc (R4) chose binary unweighted scoring for simplicity. Adding tiers as a secondary signal preserves the simple model while communicating that not all criteria are equally important. Three tiers are enough to differentiate without overcomplicating.

**Tier assignment approach**: Criteria that directly affect code correctness or agent operability (type_check, unit_tests_exist, agents_md, lint_config) are high impact. Criteria that improve quality but aren't blocking (code_modularization, duplicate_code_detection) are low impact. Everything else is medium.

**Weighted score formula**: `sum(weight * passed) / sum(weight * applicable) * 100`

**Alternatives considered**: Per-criterion numeric weights (1-10 scale) — rejected because too granular, hard to justify individual numbers. Two tiers (important/nice-to-have) — insufficient differentiation.

## R5: `/agentfit-fix` Architecture

**Decision**: Separate skill file (`agentfit-fix.md`) that reads JSON sidecar from last `/agentfit` run. Interactive output mode (user chooses commit/branch/unstaged).

**Rationale**: Separation required by Constitution IX (read-only constraint for assessment). Interactive mode respects developer workflow preferences per clarification Q3.

**Priority fixes (remediation templates)**:
1. `AGENTS.md` — extract project info from README, package manifests, CI configs
2. `.pre-commit-config.yaml` — language-appropriate hooks (detected from project language)
3. `.devcontainer/devcontainer.json` — base image from detected language
4. `.env.example` — template from existing `.env` patterns or common framework defaults
5. `dependabot.yml` — package ecosystem from detected language
6. `CODEOWNERS` — extract from git log contributor patterns
7. `.github/ISSUE_TEMPLATE/` — standard bug report and feature request templates

**Alternatives considered**: In-place remediation within `agentfit.md` — rejected because it violates read-only principle. PR-based remediation via `gh` — rejected because requires GitHub authentication and is too opinionated.

## R6: CI Integration Approach

**Decision**: Documentation-only for v2. Provide a copy-paste GitHub Action workflow YAML. No custom action published.

**Rationale**: Publishing a custom GitHub Action requires its own repo, versioning, and maintenance. A documented workflow YAML that runs Claude Code with `/agentfit` achieves the same result with zero maintenance overhead. Trend tracking via JSON artifacts is built into the workflow.

**Alternatives considered**: Custom GitHub Action (`uses: agentfit/action@v2`) — rejected per Constitution II (simplicity) and IV (zero dependencies). GitHub App integration — too complex for v2 scope.

## R7: Schema Version Strategy

**Decision**: `schema_version` in JSON output aligned with `plugin.json` version. Follows semver: MAJOR for breaking changes (criteria removed/renamed), MINOR for additive changes (new criteria), PATCH for fixes.

**Rationale**: Aligning with plugin.json avoids maintaining two version numbers. Semver communicates the nature of changes to JSON consumers.

**Alternatives considered**: Separate schema version independent of plugin version — rejected because it adds confusion and maintenance. Date-based versioning — not expressive enough about breaking vs additive changes.
