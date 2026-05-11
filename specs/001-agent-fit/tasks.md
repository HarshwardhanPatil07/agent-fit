# Tasks: Agent-Fit Assessment Skill

**Input**: Design documents from `specs/001-agent-fit/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup

**Purpose**: Project structure and plugin manifest updates

- [ ] T001 Update plugin manifest version and description to reflect 9-pillar framework in plugins/agentfit/.claude-plugin/plugin.json
- [ ] T002 [P] Update marketplace manifest to match new plugin description in .claude-plugin/marketplace.json
- [ ] T003 [P] Create examples directory at plugins/agentfit/examples/

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core skill structure that MUST be complete before any user story

**CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 Write the skill header and invocation instructions (Step 1: Discover Project Structure) in plugins/agentfit/commands/agentfit.md — detect primary language from manifest files (go.mod, package.json, pyproject.toml, Cargo.toml, CMakeLists.txt, Package.swift), extract project name from directory or git remote, extract repo path
- [ ] T005 Define all 75+ criteria with their signal types, pillar assignments, and level mappings as a structured reference table in plugins/agentfit/commands/agentfit.md — each criterion entry must include: snake_case name, pillar number, maturity level, signal type, what to check, and tools/files that satisfy it
- [ ] T006 Write the scoring methodology section in plugins/agentfit/commands/agentfit.md — pass rate calculation (criteria_passed / applicable_criteria * 100), per-pillar percentage, gated maturity level progression (80% rule), skip logic for non-applicable criteria
- [ ] T007 Write the HTML report template with dark-theme inline CSS in plugins/agentfit/commands/agentfit.md — header (project name, language badge, repo path, pass rate), level progress bar (L1-L5), summary section, strengths/opportunities columns, ALL CRITERIA section with per-pillar groups and per-criterion rows matching the visual design tokens from contracts/report-schema.md

**Checkpoint**: Foundation ready — skill structure, criteria definitions, scoring rules, and report template all defined

---

## Phase 3: User Story 1 - Run Agent-Fit Assessment (Priority: P1) MVP

**Goal**: A developer runs `/agentfit` and receives a complete HTML report with overall pass rate, maturity level, 9 pillar scores, and per-criterion findings

**Independent Test**: Run `/agentfit` in any project directory and verify a complete HTML report opens in the browser with all 9 pillars scored

### Implementation for User Story 1

- [ ] T008 [US1] Write Pillar 1 evaluation logic (Style & Validation, 13 criteria) in plugins/agentfit/commands/agentfit.md — formatter (check for .prettierrc, pyproject.toml [tool.black], rustfmt.toml, .clang-format), lint_config (eslintrc, golangci-lint, clippy, ruff), type_check, strict_typing, pre_commit_hooks (.pre-commit-config.yaml, .husky/), naming_consistency, large_file_detection (.gitattributes LFS), code_modularization (internal/ packages, eslint-plugin-boundaries), cyclomatic_complexity, dead_code_detection (knip, vulture, staticcheck), duplicate_code_detection (jscpd, PMD CPD), tech_debt_tracking (CI TODO scanning), n_plus_one_detection
- [ ] T009 [P] [US1] Write Pillar 2 evaluation logic (Build System, 19 criteria) in plugins/agentfit/commands/agentfit.md — build_cmd_doc (README/AGENTS.md/Makefile), deps_pinned (lockfiles), vcs_cli_tools (gh CLI), single_command_setup, fast_ci_feedback (CI workflow timing), deployment_frequency (git tag/release count), release_automation (release workflows), release_notes_automation (goreleaser/towncrier/changesets), automated_pr_review (bot presence), agentic_development (.claude/skills/, agent co-author in git log), feature_flag_infrastructure, build_performance_tracking, unused_deps_detection, monorepo_tooling, dead_feature_flag_detection, heavy_dependency_detection, progressive_rollout, rollback_automation, version_drift_detection
- [ ] T010 [P] [US1] Write Pillar 3 evaluation logic (Testing, 8 criteria) in plugins/agentfit/commands/agentfit.md — unit_tests_exist (*_test.go, test_*.py, *.test.ts, #[test]), unit_tests_runnable (documented test command), integration_tests_exist (Playwright, Cypress, acceptance tests), test_coverage_thresholds (codecov.yml, jest --coverage config), test_naming_conventions, test_isolation (t.Parallel(), pytest-xdist, jest workers), flaky_test_detection (rerun plugins, stress flags), test_performance_tracking (--durations flags)
- [ ] T011 [P] [US1] Write Pillar 4 evaluation logic (Documentation, 8 criteria) in plugins/agentfit/commands/agentfit.md — readme (README.md existence and comprehensiveness), agents_md (AGENTS.md or CLAUDE.md), agents_md_validation (CI job testing documented commands), documentation_freshness (git log for file modification within 180 days), automated_doc_generation (Sphinx, godoc, MkDocs, Typedoc workflows), api_schema_docs (OpenAPI, .proto, GraphQL schema files), service_flow_documented (PlantUML, Mermaid, architecture diagrams), skills (.claude/skills/ or .factory/skills/ directories)
- [ ] T012 [P] [US1] Write Pillar 5 evaluation logic (Development Environment, 5 criteria) in plugins/agentfit/commands/agentfit.md — devcontainer (.devcontainer/devcontainer.json), devcontainer_runnable (execution test), env_template (.env.example, .envrc.example), local_services_setup (docker-compose.yml with services), database_schema (migration files, schema.prisma, SQLAlchemy models)
- [ ] T013 [P] [US1] Write Pillar 6 evaluation logic (Debugging & Observability, 11 criteria) in plugins/agentfit/commands/agentfit.md — structured_logging (zap, logrus, structlog, Winston imports), distributed_tracing (OpenTelemetry, Jaeger, X-Request-ID), metrics_collection (Prometheus, Datadog, OpenTelemetry metrics), alerting_configured (Alertmanager, PagerDuty, OpsGenie), deployment_observability (Grafana dashboards, monitoring config), error_tracking_contextualized (Sentry, Bugsnag, Rollbar), health_checks (/health endpoint, liveness probes), profiling_instrumentation (pprof, py-spy, Pyroscope), code_quality_metrics (Codecov, CodeQL, SonarQube), circuit_breakers (resilience libraries), runbooks_documented (runbooks/ directory, INCIDENT_RESPONSE.md)
- [ ] T014 [P] [US1] Write Pillar 7 evaluation logic (Security & Governance, 11 criteria) in plugins/agentfit/commands/agentfit.md — branch_protection (gh api rulesets, skip if gh unavailable), codeowners (.github/CODEOWNERS), secret_scanning (gh api, skip if unavailable), secrets_management (vault references, secrets.* patterns, GitHub Actions secrets), dependency_update_automation (dependabot.yml, renovate.json), gitignore_comprehensive (.gitignore covers .env, credentials, build artifacts), automated_security_review (CodeQL, Snyk, Trivy in CI), log_scrubbing (redaction patterns, sanitization), pii_handling (data classification), dast_scanning (OWASP ZAP in CI), privacy_compliance (GDPR tooling)
- [ ] T015 [P] [US1] Write Pillar 8 evaluation logic (Task Discovery, 4 criteria) in plugins/agentfit/commands/agentfit.md — issue_templates (.github/ISSUE_TEMPLATE/), issue_labeling_system (gh api labels, skip if unavailable), backlog_health (gh api issues, check title length and label coverage, skip if unavailable), pr_templates (.github/pull_request_template.md or PULL_REQUEST_TEMPLATE/)
- [ ] T016 [P] [US1] Write Pillar 9 evaluation logic (Product & Experimentation, 3 criteria) in plugins/agentfit/commands/agentfit.md — product_analytics_instrumentation (Mixpanel, Amplitude, PostHog, GA4 imports), error_to_insight_pipeline (Sentry-GitHub integration, automated issue creation), experiment_infrastructure (feature flag with metrics, A/B framework)
- [ ] T017 [US1] Write the assessment orchestration section in plugins/agentfit/commands/agentfit.md — tie all 9 pillar evaluations together: run all checks, collect results, compute per-pillar scores, compute overall pass rate, determine maturity level via gated progression, select top 3 strengths and opportunities, generate narrative summary headline, render HTML report, write to temp file, open in browser

**Checkpoint**: User Story 1 fully functional — `/agentfit` produces a complete HTML report with all 9 pillars scored

---

## Phase 4: User Story 2 - Understand Per-Criterion Findings (Priority: P2)

**Goal**: Every criterion finding includes specific evidence — file paths for FOUND, tool names for MISSING, skip rationale for skipped

**Independent Test**: Review all criterion evidence strings and verify each FOUND references a specific file/config, each MISSING names what to create, and each skipped explains why

### Implementation for User Story 2

- [ ] T018 [US2] Review and enhance all 75+ criterion evidence templates in plugins/agentfit/commands/agentfit.md — ensure every FOUND evidence references the actual file path or config value detected (e.g., "golangci-lint configured with errcheck, staticcheck, revive in .golangci.yml"), every MISSING evidence specifies what to create (e.g., "No .pre-commit-config.yaml, husky, or similar pre-commit hook framework found"), and every skipped evidence explains why (e.g., "Skipped - library project without database access, N+1 detection not applicable")
- [ ] T019 [US2] Add language-specific evidence detail in plugins/agentfit/commands/agentfit.md — for each criterion, include language-appropriate tool names: Go (golangci-lint, gofumpt, go test -race), Python (mypy, Black, Ruff, pytest), TypeScript (ESLint, Prettier, Jest), Rust (Clippy, rustfmt, cargo test), C++ (clang-format, clang-tidy), Swift (SwiftLint, XCTest)

**Checkpoint**: Every criterion produces specific, actionable evidence strings

---

## Phase 5: User Story 3 - Prioritized Remediation Recommendations (Priority: P3)

**Goal**: Report includes remediation recommendations ordered by maturity level progression — fix Level 1 gaps before Level 2, Level 2 before Level 3

**Independent Test**: Verify the strengths/opportunities section prioritizes by level and the overall ordering makes sense for a developer with limited time

### Implementation for User Story 3

- [ ] T020 [US3] Write remediation prioritization logic in plugins/agentfit/commands/agentfit.md — sort MISSING criteria by maturity level (L1 gaps first, then L2, L3, L4, L5), within each level sort by pillar pass rate impact, select top 3 for the Opportunities section, and include the remediation action (e.g., "Set up pre-commit hooks (Husky, pre-commit) to catch issues before they reach CI and speed up feedback loops")
- [ ] T021 [US3] Write strength selection logic in plugins/agentfit/commands/agentfit.md — select top 3 highest-scoring pillars for the Strengths section, include the pillar percentage and key passing criteria as evidence (e.g., "Testing (100%) — Includes Flaky Test Detection, Integration Tests Exist, Test Coverage Thresholds")

**Checkpoint**: Report shows level-ordered opportunities and evidence-backed strengths

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, validation, and examples

- [ ] T022 [P] Update plugins/agentfit/README.md — reflect 9-pillar framework, 0-100% scoring, maturity levels, HTML report output, supported languages, optional gh CLI dependency
- [ ] T023 [P] Create sample HTML report at plugins/agentfit/examples/sample-report.html — a static example showing a sample assessment (e.g., the Temporal report from the screenshots) for reference
- [ ] T024 Validate skill by running /agentfit on the agent-fit repository itself — verify report generates, all 9 pillars produce results, HTML opens in browser, scores are reasonable
- [ ] T025 [P] Validate skill on a Go repository (e.g., clone a public Go repo) — verify language detection, Go-specific criteria (go.mod, *_test.go, internal/), and scoring
- [ ] T026 [P] Validate skill on a Python repository — verify language detection, Python-specific criteria (pyproject.toml, pytest, mypy), and scoring
- [ ] T027 [P] Validate skill on a TypeScript repository — verify language detection, TS-specific criteria (tsconfig.json, ESLint, Jest), and scoring
- [ ] T028 Run quickstart.md validation — follow the quickstart guide end-to-end and verify it works as documented

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Phase 2 — pillar evaluations can run in parallel
- **User Story 2 (Phase 4)**: Depends on Phase 3 (US1 must be working first)
- **User Story 3 (Phase 5)**: Depends on Phase 3 (US1 must be working first)
- **Polish (Phase 6)**: Depends on all user stories being complete

### Within User Story 1

- T008-T016 (pillar evaluations) are ALL parallelizable — they write to different sections of the same file but cover independent criteria sets
- T017 (orchestration) depends on ALL pillar evaluations being complete

### User Story 2 and 3 Independence

- US2 (T018-T019) and US3 (T020-T021) can run in parallel after US1 is complete
- They modify different sections of agentfit.md (evidence templates vs. prioritization logic)

### Parallel Opportunities

```bash
# Phase 1: All setup tasks in parallel
T001, T002, T003

# Phase 2: T005, T006, T007 can overlap after T004
T004 → T005 [P], T006 [P], T007 [P]

# Phase 3 (US1): All pillar evaluations in parallel
T008, T009, T010, T011, T012, T013, T014, T015, T016
→ T017 (orchestration, depends on all above)

# Phase 4-5: US2 and US3 in parallel
T018, T019 (US2) || T020, T021 (US3)

# Phase 6: Validation tasks in parallel
T022, T023 [P] → T024, T025, T026, T027 [P] → T028
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T003)
2. Complete Phase 2: Foundational (T004-T007)
3. Complete Phase 3: User Story 1 (T008-T017)
4. **STOP and VALIDATE**: Run `/agentfit` on agent-fit repo, verify HTML report
5. If working: proceed to US2 + US3

### Incremental Delivery

1. Setup + Foundational → Skill structure ready
2. Add US1 → Run assessment, get HTML report (MVP!)
3. Add US2 → Evidence quality improved
4. Add US3 → Remediation prioritization added
5. Polish → Documentation, examples, cross-repo validation

---

## Notes

- [P] tasks = different files or different sections, no dependencies
- [Story] label maps task to specific user story for traceability
- All implementation happens in a single file: plugins/agentfit/commands/agentfit.md
- The file is a Claude Code skill definition (markdown instructions), not executable code
- Each task adds/modifies a specific section of that file
- T017 is the integration point — ties all pillar evaluations into the orchestration flow
