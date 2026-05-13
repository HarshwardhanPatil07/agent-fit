# Feature Specification: Agent-Fit v2 Architecture & Future

**Feature Branch**: `002-agent-fit-v2-architecture`
**Created**: 2026-05-13
**Status**: Draft
**Input**: User description: "Architecture & Future improvements for agent-fit v2. This includes: (1) Project-type profiles to centralize skip rules, (2) Automated remediation command /agentfit-fix, (3) Custom criteria extension via .agentfit.yml, (4) Weighted scoring as optional secondary signal, (5) CI integration and GitHub Action, (6) Skill versioning and schema evolution, (7) Complete validation tasks T024-T028 on real repos."

## Clarifications

### Session 2026-05-13

- Q: When a project matches multiple types (e.g., monorepo + CLI + web app), how should the applicability matrix resolve the conflict? → A: Union of all matched types — skip a criterion only if ALL matched types would skip it.
- Q: When `.agentfit.yml` disables a criterion that the project-type matrix would evaluate (or vice versa), which takes precedence? → A: `.agentfit.yml` always wins — user config overrides project-type matrix in both directions.
- Q: When `/agentfit-fix` generates remediation files, how should it stage the changes? → A: Interactive mode — generates files, then asks the user whether to commit to current branch, create a new branch, or leave unstaged.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Project-Type-Aware Assessment (Priority: P1)

A developer runs `/agentfit` on a CLI tool repository. The skill detects the project type (CLI) in Step 1 and automatically skips criteria that don't apply — DAST scanning, health checks, distributed tracing, deployment observability, and product analytics. The report shows these as "Skipped — CLI project" instead of penalizing the score. The same developer runs `/agentfit` on a web app and sees all criteria evaluated.

**Why this priority**: Without centralized project-type detection, skip rules are scattered per-criterion with inconsistent coverage. A CLI project that scores 45% because it's penalized for missing health checks is misleading. This directly impacts score accuracy for every non-web-app project.

**Independent Test**: Run `/agentfit` on a CLI project (e.g., a Go CLI with `cobra` or a Python CLI with `click`) and verify that DAST, health_checks, distributed_tracing, deployment_observability, and product_analytics are all marked as skipped with "CLI project" rationale. Then run on a web app and verify all criteria are evaluated.

**Acceptance Scenarios**:

1. **Given** a Go project with only a `main.go` and CLI flags (no HTTP server, no web framework), **When** the user runs `/agentfit`, **Then** the project type is detected as "CLI" and at least 5 non-applicable criteria are skipped with a clear rationale.
2. **Given** a Python project with `flask` or `fastapi` in its dependencies, **When** the user runs `/agentfit`, **Then** the project type is detected as "Web App" or "API Service" and all criteria are evaluated (none skipped due to project type).
3. **Given** a TypeScript project with only `package.json` exporting library modules (no `bin` field, no server framework), **When** the user runs `/agentfit`, **Then** the project type is detected as "Library" and deployment-related criteria (deployment_frequency, progressive_rollout, rollback_automation, health_checks, product_analytics, local_services_setup, DAST) are skipped.
4. **Given** a project with `pnpm-workspace.yaml` or `turbo.json`, **When** the user runs `/agentfit`, **Then** the project type is detected as "Monorepo" and monorepo-specific criteria (monorepo_tooling, version_drift_detection) are NOT skipped.
5. **Given** a monorepo containing both a CLI tool and a web app (detected types: monorepo + CLI + web_app), **When** the user runs `/agentfit`, **Then** only criteria that ALL three types would skip are actually skipped (union approach). For example, product_analytics is not skipped because web_app evaluates it, even though CLI would skip it.

---

### User Story 2 - Skill Versioning and Schema Evolution (Priority: P1)

A developer runs `/agentfit` and the generated HTML report and JSON output both include the skill version number (from plugin.json) and a schema version identifier. When the criteria set changes in a future release, consumers of the JSON output can detect the schema change and update their tooling accordingly.

**Why this priority**: The criteria set will evolve — criteria will be added, removed, or reclassified. Without schema versioning, JSON consumers (CI integrations, dashboards, trend trackers) will silently break when the output format changes. This is a prerequisite for CI integration (US5).

**Independent Test**: Run `/agentfit`, inspect the HTML report footer for skill version and the JSON output for `schema_version`. Verify both match the version in `plugin.json`.

**Acceptance Scenarios**:

1. **Given** `plugin.json` has `"version": "2.0.0"`, **When** the user runs `/agentfit`, **Then** the HTML report footer displays "Agent-Fit v2.0.0" and the JSON output includes `"schema_version": "2.0.0"`.
2. **Given** a new criterion is added and the version is bumped to 2.1.0, **When** a consumer compares JSON outputs from v2.0.0 and v2.1.0, **Then** the `schema_version` field differs, signaling a schema change.
3. **Given** the HTML report is generated, **When** the user scrolls to the bottom, **Then** a footer section displays: assessment date, skill version, total criteria evaluated, criteria skipped, git SHA of the assessed repo.

---

### User Story 3 - Custom Criteria Extension (Priority: P2)

A team maintains a `.agentfit.yml` file at their repo root that defines two additional project-specific criteria ("internal_api_docs" and "staging_environment") and disables one irrelevant criterion ("product_analytics_instrumentation"). When `/agentfit` runs, it includes the custom criteria in the assessment and excludes the disabled one.

**Why this priority**: Different teams have different quality standards. A fintech team may require specific compliance checks that aren't in the default set. A library team may want to disable criteria that will never apply. This makes agent-fit adaptable without forking the skill.

**Independent Test**: Create a `.agentfit.yml` with one custom criterion and one disabled criterion, run `/agentfit`, and verify the custom criterion appears in the report and the disabled one is absent.

**Acceptance Scenarios**:

1. **Given** a `.agentfit.yml` with `custom_criteria: [{name: "internal_api_docs", pillar: "Documentation", level: "L2", check: "docs/api/ directory exists", found_when: "docs/api/ contains at least one file"}]`, **When** the user runs `/agentfit` in a repo with `docs/api/endpoints.md`, **Then** the report includes "internal_api_docs" under the Documentation pillar marked as FOUND.
2. **Given** a `.agentfit.yml` with `disabled_criteria: ["product_analytics_instrumentation"]`, **When** the user runs `/agentfit`, **Then** "product_analytics_instrumentation" does not appear in the report at all (not found, not missing, not skipped — fully removed).
3. **Given** a malformed `.agentfit.yml` (invalid YAML), **When** the user runs `/agentfit`, **Then** the skill warns "Invalid .agentfit.yml — skipping custom criteria" and proceeds with the default criteria set.

---

### User Story 4 - Weighted Scoring (Priority: P2)

A developer runs `/agentfit` and alongside the existing binary pass rate (e.g., "67%"), the report shows a weighted score (e.g., "72%") based on impact tiers. High-impact criteria like type_check and unit_tests_exist count more than low-impact criteria like duplicate_code_detection. The weighted score is displayed as a secondary signal, not a replacement.

**Why this priority**: Not all criteria are equally important. A project with type checking and tests but missing duplicate code detection is fundamentally healthier than the reverse. Weighted scoring communicates this nuance without breaking the existing simple scoring model.

**Independent Test**: Run `/agentfit` on a project that passes high-impact criteria but fails low-impact ones. Verify the weighted score is higher than the unweighted pass rate.

**Acceptance Scenarios**:

1. **Given** a project passing type_check (high) and unit_tests_exist (high) but failing duplicate_code_detection (low) and tech_debt_tracking (low), **When** the report is generated, **Then** the weighted score is higher than the unweighted pass rate.
2. **Given** the report is generated, **When** the user views the header section, **Then** both the unweighted pass rate and weighted score are displayed, with the weighted score labeled as "Weighted Score" and a tooltip or note explaining what it means.
3. **Given** no `.agentfit.yml` with custom weights, **When** `/agentfit` runs, **Then** default impact tiers are used (high: 3x, medium: 2x, low: 1x).

---

### User Story 5 - CI Integration and GitHub Action (Priority: P3)

A team adds agent-fit to their GitHub Actions CI pipeline. On every pull request, the action runs `/agentfit`, stores the JSON output as a build artifact, and posts a summary comment on the PR showing the pass rate, maturity level, and any criteria that changed status since the last run.

**Why this priority**: Continuous assessment is more valuable than one-off checks. CI integration enables trend tracking and prevents agent-readiness regressions. However, this depends on stable JSON output (US2) and is lower priority than getting the core assessment right.

**Independent Test**: Create a GitHub Action workflow that runs agent-fit and verify: (a) JSON artifact is uploaded, (b) PR comment is posted with pass rate and maturity level.

**Acceptance Scenarios**:

1. **Given** a repository with `.github/workflows/agentfit.yml` configured per the documentation, **When** a pull request is opened, **Then** the action runs the assessment and uploads a JSON artifact named `agentfit-report-{sha}.json`.
2. **Given** a CI run completes, **When** the PR is viewed on GitHub, **Then** a comment from the action shows: maturity level, pass rate, and a link to the full HTML report artifact.
3. **Given** a previous CI run exists for the base branch, **When** the current run completes, **Then** the PR comment includes deltas: "Pass rate: 67% (+5%)" and lists any criteria that changed from MISSING to FOUND or vice versa.

---

### User Story 6 - Automated Remediation (Priority: P3)

A developer runs `/agentfit-fix` after reviewing their assessment report. The command reads the JSON output from the last `/agentfit` run and generates files for the top N missing criteria — for example, creating an `AGENTS.md` template, a `.pre-commit-config.yaml` with language-appropriate hooks, a `.devcontainer/devcontainer.json`, an `.env.example`, a `dependabot.yml`, and issue/PR templates. Changes are staged as a new commit or branch for review.

**Why this priority**: Explicitly deferred from v1 per spec.md ("Report only (read-only) for v1"). This is the natural next step after assessment — turning findings into action. Lower priority because it requires stable assessment output first.

**Independent Test**: Run `/agentfit`, then run `/agentfit-fix`, and verify that at least one missing criterion now has an appropriate file generated.

**Acceptance Scenarios**:

1. **Given** an agentfit JSON report showing AGENTS.md as MISSING, **When** the user runs `/agentfit-fix`, **Then** an `AGENTS.md` file is created with project-appropriate content (build commands, test instructions, coding conventions extracted from the codebase).
2. **Given** an agentfit JSON report with 10 missing criteria, **When** the user runs `/agentfit-fix --top 3`, **Then** only the 3 highest-priority missing criteria are remediated (prioritized by maturity level, L1 first).
3. **Given** the user runs `/agentfit-fix`, **When** files are generated, **Then** the command presents an interactive prompt asking the user to choose: (a) commit to current branch, (b) create a new branch `agentfit-fix/{date}` with a commit, or (c) leave files unstaged for manual review.

---

### User Story 7 - Validation on Real Repositories (Priority: P1)

A maintainer runs `/agentfit` against 5 real repositories (agent-fit itself, a Go project, a Python project, a TypeScript project, and a quickstart template) and verifies end-to-end correctness: the report generates without errors, all 9 pillars produce results, scores are reasonable, and no criteria produce false positives or false negatives.

**Why this priority**: The skill is implemented but unvalidated end-to-end (tasks T024-T028 are incomplete). Shipping without validation risks inaccurate assessments that undermine user trust. This gates the v1 release.

**Independent Test**: Run `/agentfit` on each of the 5 target repos and compare results against manual inspection of each repo's actual tooling.

**Acceptance Scenarios**:

1. **Given** the agent-fit repository itself, **When** `/agentfit` is run, **Then** the report generates successfully, shows the correct language (TypeScript/Markdown), and pillar scores are consistent with the repo's actual tooling (has pre-commit hooks, has tests, has CLAUDE.md).
2. **Given** a public Go repository (e.g., one with go.mod, *_test.go files, and CI), **When** `/agentfit` is run, **Then** Go-specific criteria (go.mod detection, *_test.go detection, golangci-lint) are correctly evaluated.
3. **Given** a Python repository with mypy, pytest, and Black configured, **When** `/agentfit` is run, **Then** the corresponding criteria (type_check, unit_tests_exist, formatter) are all marked FOUND with evidence citing the specific config files.
4. **Given** a TypeScript repository with ESLint, Jest, and tsconfig.json, **When** `/agentfit` is run, **Then** TypeScript-specific criteria are correctly evaluated.
5. **Given** any validation run, **When** the report is compared to manual inspection, **Then** fewer than 5% of criteria produce false positives or false negatives.

---

### Edge Cases

- ~~What happens when a project matches multiple types?~~ **Resolved**: Union of all matched types — skip only if ALL types would skip. A monorepo containing a CLI and a web app evaluates the union of both type matrices.
- ~~How does `.agentfit.yml` interact with project-type skip rules?~~ **Resolved**: `.agentfit.yml` always wins — user config overrides project-type matrix in both directions (can force-evaluate skipped criteria or disable evaluated ones).
- What happens when `/agentfit-fix` is run but no previous JSON report exists? The command MUST error with a clear message: "No agentfit report found. Run /agentfit first."
- How does weighted scoring handle skipped criteria? Skipped criteria are excluded from both the unweighted and weighted calculations (they don't count toward total or passed).
- What happens when a CI run fails mid-assessment (partial JSON output)? The GitHub Action should treat incomplete JSON as a failed run and not post a PR comment.
- How does the GitHub Action handle repos with no previous baseline (first run, no delta to show)? Show absolute scores only, no delta column. Note: "First assessment — no baseline for comparison."

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST detect project type in Step 1 as one or more of: library, CLI, web app, API service, or monorepo, based on manifest files, dependency analysis, and directory structure signals. Multiple types can be detected simultaneously.
- **FR-002**: System MUST define a centralized applicability matrix mapping each project type to the criteria that should be skipped. When multiple types are detected, a criterion is skipped only if ALL detected types would skip it (union approach). `.agentfit.yml` overrides the matrix in both directions.
- **FR-003**: System MUST include `schema_version` in JSON output and skill version in both JSON and HTML output, sourced from `plugin.json`.
- **FR-004**: System MUST include a report footer in the HTML output with: assessment date, skill version, total criteria evaluated, criteria skipped, and git SHA of the assessed repo.
- **FR-005**: System MUST read `.agentfit.yml` from the repo root (if present) to load custom criteria definitions and disabled criteria lists.
- **FR-006**: System MUST validate `.agentfit.yml` against a defined schema and warn (not error) on invalid content.
- **FR-007**: System MUST compute a weighted score using impact tiers (high=3x, medium=2x, low=1x) and display it alongside the unweighted pass rate.
- **FR-008**: System MUST provide documentation for running agent-fit in a GitHub Action, including workflow YAML examples.
- **FR-009**: System MUST support JSON artifact storage per CI run for trend tracking.
- **FR-010**: The `/agentfit-fix` command MUST be a separate skill file that reads JSON output and generates files for missing criteria.
- **FR-011**: The `/agentfit-fix` command MUST support a `--top N` parameter to limit remediation to the N highest-priority missing criteria.
- **FR-012**: System MUST be validated end-to-end against at least 5 real repositories before the v2 release is declared shipped.
- **FR-013**: System MUST bump `plugin.json` version when criteria are added, removed, or reclassified.

### Key Entities

- **Project Type**: One of {library, CLI, web_app, api_service, monorepo}. Determined by analyzing manifest files, dependencies, and directory structure. Drives the applicability matrix.
- **Applicability Matrix**: A mapping from project type to a set of criteria names that should be skipped. Replaces scattered per-criterion skip rules.
- **Custom Criteria**: User-defined assessment criteria specified in `.agentfit.yml`. Each has: name, pillar, level, check description, and found_when condition.
- **Impact Tier**: A classification of criteria importance (high, medium, low) used to compute weighted scores. Each tier has a multiplier (3x, 2x, 1x).
- **Schema Version**: A version identifier included in JSON output that changes when the criteria set or output format changes. Follows semver aligned with plugin.json.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A CLI project and a library project both score within 5% of their "true" agent-readiness when irrelevant criteria are correctly skipped (measured by manual inspection of skip correctness).
- **SC-002**: The JSON output includes `schema_version` and the HTML report includes a footer with skill version, assessment date, and git SHA on every run.
- **SC-003**: A `.agentfit.yml` with custom criteria produces a report that includes those criteria with correct FOUND/MISSING status on the first run without errors.
- **SC-004**: The weighted score differs from the unweighted pass rate on at least 80% of assessed repositories (demonstrating the signal adds information).
- **SC-005**: A GitHub Action workflow example can be copy-pasted into a repo and produces a valid JSON artifact on the first CI run.
- **SC-006**: `/agentfit-fix` generates at least 3 valid, project-appropriate files from a single run on a repo with multiple missing criteria.
- **SC-007**: End-to-end validation on 5 real repos produces fewer than 5% false positive/negative rate across all criteria.

## Assumptions

- The existing v1 skill (`plugins/agentfit/commands/agentfit.md`) is the foundation — v2 extends it rather than rewriting from scratch.
- Project-type detection can be done reliably from manifest files and dependency analysis without running the project's build system.
- The `.agentfit.yml` schema will be simple enough that YAML parsing within the skill instructions is sufficient (no external schema validator needed).
- Impact tier assignments (high/medium/low) for existing criteria will be determined by the team based on benchmarking data from the 37-repo analysis.
- CI integration documentation targets GitHub Actions specifically; other CI systems are out of scope for v2.
- The `/agentfit-fix` command is write-mode (it creates files), which is a deliberate departure from v1's read-only constraint. It will be a separate skill file with its own permissions.
- Validation repos will be publicly available repositories that can be cloned without authentication.
