---
description: Evaluates codebase agent-readiness across 9 pillars, 75+ criteria, and 5 maturity levels — produces an HTML report with pass rate scoring
---

When invoked, perform the following assessment. This is a READ-ONLY analysis — do NOT modify any files in the target codebase. The only file you create is the HTML report.

## Step 1: Discover Project Context

1. List root directory files and folders.
2. Detect the primary language by checking for these manifest files (first match wins):
   - `go.mod` → Go
   - `Cargo.toml` → Rust
   - `pyproject.toml` or `setup.py` or `requirements.txt` → Python
   - `package.json` with `tsconfig.json` → TypeScript
   - `package.json` without `tsconfig.json` → JavaScript
   - `CMakeLists.txt` or `Makefile` with `.cpp`/`.cc`/`.h` files → C++
   - `Package.swift` → Swift
   - If none found → Unknown
3. Extract project name: use the current directory name.
4. Extract repo path: run `git remote get-url origin 2>/dev/null` and parse `org/repo` from the URL. If no git remote, use the directory path.
5. Get a short project description from README.md first line/paragraph if available.
6. The skill version is `2.0.0`. Store as `skill_version` for use in the report footer and JSON output.
7. Get the git SHA: run `git rev-parse --short HEAD 2>/dev/null`. Store as `git_sha`.
8. Detect project type(s). A project can match multiple types. Check all of the following and record every match:
   - **library**: `package.json` with no `bin` field and no server framework (`express`, `fastify`, `koa`, `hapi`, `next`, `nuxt`) in dependencies; OR `pyproject.toml`/`setup.py` with no CLI `[project.scripts]` entry_points and no web framework; OR Go project with no `main.go` or with only exported packages; OR Rust project with `[lib]` in Cargo.toml and no `[[bin]]`
   - **cli**: `package.json` with `bin` field; OR `pyproject.toml` with `[project.scripts]`; OR `main.go` with `cobra`, `urfave/cli`, or `flag` imports; OR Cargo.toml with `[[bin]]` and no HTTP framework
   - **web_app**: HTTP framework in dependencies (`react`, `next`, `vue`, `nuxt`, `angular`, `svelte`, `flask`, `django`, `rails`) AND presence of frontend assets (`public/`, `static/`, `templates/`, HTML/CSS source files)
   - **api_service**: HTTP framework in dependencies (`fastapi`, `gin`, `echo`, `express`, `actix-web`, `axum`, `flask`, `django-rest-framework`) WITHOUT frontend assets (no `public/`, `static/`, `templates/` with HTML)
   - **monorepo**: `pnpm-workspace.yaml`, `lerna.json`, `nx.json`, `turbo.json`, Cargo workspace `[workspace]` in Cargo.toml, or multiple `go.mod` files
   - If no type matches, default to `web_app` (most permissive — evaluates all criteria)
   - Record detection evidence for each matched type (e.g., "Found `bin` field in package.json")
9. Read `.agentfit.yml` from the repo root (if it exists):
   - Parse the YAML file. If parsing fails (invalid YAML), warn: "Invalid .agentfit.yml — skipping custom criteria" and proceed with defaults.
   - Extract `custom_criteria` list (if present). For each entry, validate: `name` must be snake_case and unique, `pillar` must match one of the 9 pillar names, `level` must be L1-L5. Skip invalid entries with a warning.
   - Extract `disabled_criteria` list (if present). These criterion names will be fully removed from the assessment.
   - `.agentfit.yml` overrides the project-type applicability matrix in BOTH directions: `disabled_criteria` can force-remove criteria the matrix would evaluate; custom criteria are always evaluated regardless of project type.

## Step 2: Evaluate All Criteria

Evaluate each criterion below using the specified signal type. For each criterion, record:
- **status**: `found` (passes), `missing` (fails), or `skipped` (not applicable)
- **score**: `1/1`, `0/1`, or `—/—`
- **evidence**: A specific sentence referencing the actual file/config found, or what is missing, or why it was skipped

### Criteria Skip Rules

**Project-Type Applicability Matrix**: Skip a criterion (mark as `—/—`) based on the detected project type(s). When multiple types are detected, skip a criterion ONLY if ALL detected types would skip it (union approach).

| Criterion | Library | CLI | Web App | API Service | Monorepo |
|-----------|---------|-----|---------|-------------|----------|
| deployment_frequency | SKIP | — | — | — | — |
| progressive_rollout | SKIP | — | — | — | — |
| rollback_automation | SKIP | — | — | — | — |
| health_checks | SKIP | SKIP | — | — | — |
| product_analytics_instrumentation | SKIP | SKIP | — | SKIP | — |
| local_services_setup | SKIP | — | — | — | — |
| dast_scanning | SKIP | SKIP | — | — | — |
| distributed_tracing | — | SKIP | — | — | — |
| deployment_observability | — | SKIP | — | — | — |
| heavy_dependency_detection | — | — | — | SKIP | — |
| n_plus_one_detection | SKIP | SKIP | — | — | — |

`—` means the criterion is evaluated (not skipped) for that project type.

**Additional skip rules** (apply regardless of project type):
- Skip if the criterion requires `gh` CLI but `gh auth status` fails or `gh` is not installed
- Skip if the criterion checks for database tooling but the project has no database (no migration files, no ORM, no schema files)

**`.agentfit.yml` overrides**: If a criterion appears in `disabled_criteria`, remove it entirely (do not show as skipped — remove from the criteria list and total count). This overrides the matrix in both directions.

When skipping, always explain why in the evidence field (e.g., "Skipped — CLI project", "Skipped — gh CLI not available").

### Pillar 1: Style & Validation (13 criteria)

Evaluate each criterion. For all config-parsing criteria, read the actual config file and report what rules/settings are configured.

1. **formatter** (config_parsing) — Check for: `.prettierrc*`, `biome.json`, `pyproject.toml` with `[tool.black]` or `[tool.ruff.format]`, `rustfmt.toml`, `.clang-format`, `.editorconfig` with formatting rules, `gofumpt`/`goimports` in Makefile or CI. FOUND if any formatter is configured.

2. **lint_config** (config_parsing) — Check for: `.eslintrc*`, `eslint.config.*`, `biome.json` with linter, `.golangci.yml`, `pyproject.toml` with `[tool.ruff]` or `[tool.pylint]`, `clippy.toml` or `Cargo.toml` with clippy config, `.clang-tidy`. FOUND if linter configured with specific rules.

3. **type_check** (config_parsing) — Check for: `tsconfig.json`, `mypy.ini` or `pyproject.toml` with `[tool.mypy]`, Go compiler (always passes for Go), Rust compiler (always passes for Rust), `pyrightconfig.json`. FOUND if type checking is configured.

4. **strict_typing** (config_parsing) — Check for: `tsconfig.json` with `"strict": true`, `mypy` with `strict = true` or `disallow_untyped_defs = true`, Go (always strict), Rust (always strict), Clippy pedantic. FOUND if strict mode is enabled.

5. **pre_commit_hooks** (file_existence) — Check for: `.pre-commit-config.yaml`, `.husky/` directory, `.lefthook.yml`, `lint-staged` in `package.json`, `scripts/pre-commit*`. FOUND if any pre-commit hook framework is configured.

6. **naming_consistency** (config_parsing + doc_content) — Check for: ESLint `naming-convention` rule, `revive` linter naming rules in golangci-lint, documented naming conventions in AGENTS.md/CLAUDE.md/CONTRIBUTING.md, `.editorconfig`. FOUND if naming conventions are documented or enforced via linter.

7. **large_file_detection** (file_existence) — Check for: `.gitattributes` with LFS entries, `pre-commit` hook `check-added-large-files`, CI jobs checking file size. FOUND if large file detection is configured.

8. **code_modularization** (config_parsing) — Check for: Go `internal/` packages, `eslint-plugin-boundaries`, Nx boundaries config, Rust module visibility, monorepo workspace boundaries. FOUND if module boundary enforcement exists.

9. **cyclomatic_complexity** (config_parsing) — Check for: `gocyclo` or `cyclop` in golangci-lint config, ESLint `complexity` rule, Ruff `C901`, Clippy `cognitive_complexity`. FOUND if complexity analysis tools are configured.

10. **dead_code_detection** (config_parsing) — Check for: `knip` in package.json, `vulture` in Python config, `staticcheck` unused checks, Ruff `F401`/`F841`, Clippy `dead_code` warnings, `deadcode` tool. FOUND if dead code detection is configured.

11. **duplicate_code_detection** (config_parsing) — Check for: `jscpd` config or in CI, `PMD CPD`, `Simian`, golangci-lint `dupl` linter, SonarQube duplication. FOUND if duplication scanner is configured.

12. **tech_debt_tracking** (ci_workflow) — Check for: CI workflows scanning for TODO/FIXME markers, `godox` linter in golangci-lint, SonarQube, `check-todos` scripts. FOUND if tech debt tracking is automated.

13. **n_plus_one_detection** (config_parsing) — Check for: `bullet` gem (Ruby), `django-auto-prefetch` or `nplusone` (Python), Prisma query warnings, ORM query analyzers. Check applicability matrix for skip. FOUND if N+1 detection is configured.

### Pillar 2: Build System (19 criteria)

1. **build_cmd_doc** (doc_content) — Check README.md, AGENTS.md, CLAUDE.md, Makefile, CONTRIBUTING.md for documented build/install/run commands. FOUND if build commands are clearly documented.

2. **deps_pinned** (file_existence) — Check for: `go.sum`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `uv.lock`, `poetry.lock`, `Cargo.lock`, `Pipfile.lock`. FOUND if lockfile exists. Note: Rust libraries intentionally gitignore Cargo.lock — skip for Rust library projects.

3. **vcs_cli_tools** (dependency_check) — Run `gh --version 2>/dev/null`. FOUND if gh CLI is installed.

4. **single_command_setup** (doc_content) — Check README.md, AGENTS.md, CONTRIBUTING.md for a single setup command (e.g., `make setup`, `docker-compose up`, `npm install`, `./dev doctor`). FOUND if single-command setup is documented.

5. **fast_ci_feedback** (ci_workflow) — Read `.github/workflows/*.yml`. Check if any CI workflow completes essential checks. Estimate based on job count and step complexity. FOUND if CI appears to complete under 10 minutes.

6. **deployment_frequency** (git_history) — Run `git tag --sort=-creatordate | head -20` and check dates. FOUND if multiple releases per month or regular release cadence visible.

7. **release_automation** (ci_workflow) — Check for: `.github/workflows/release*`, `goreleaser.yml`, publish workflows, tag-triggered release workflows. FOUND if releases are automated via CI.

8. **release_notes_automation** (config_parsing) — Check for: `.goreleaser.yml` with changelog, `towncrier.toml`, `.changeset/`, `git-cliff.toml`, release-changelog-builder action in CI. FOUND if release notes are auto-generated.

9. **automated_pr_review** (ci_workflow + code_search) — Check for: CodeRabbit, Copilot Code Review, Danger.js, custom review bots in CI, Semgrep in PR checks. FOUND if automated review comments are generated on PRs.

10. **agentic_development** (file_existence + git_history) — Check for: `.claude/skills/`, `.factory/skills/`, `AGENTS.md`, `CLAUDE.md`. Also run `git log --format="%aN" -100 2>/dev/null` and check for agent co-author signatures (Claude, Copilot, factory-droid). FOUND if agent tooling directories exist or agent co-authorship visible.

11. **feature_flag_infrastructure** (config_parsing + dependency_check) — Check for: LaunchDarkly, Statsig, Unleash, custom feature flag config files, `crates/feature_flags`. FOUND if feature flag system is configured.

12. **build_performance_tracking** (config_parsing) — Check for: Bazel distributed cache config, turbo/nx caching, sccache, build metrics export, CI build timing. FOUND if build times are cached or tracked.

13. **unused_deps_detection** (ci_workflow + config_parsing) — Check for: `depcheck` in scripts, `go mod tidy` in CI, `cargo-udeps`, `deptry`, `knip` dependency mode. FOUND if unused deps are detected automatically.

14. **monorepo_tooling** (config_parsing) — Check for: Lerna, Nx, Turborepo configs, Bazel BUILD files, Cargo workspace members, pnpm-workspace.yaml. FOUND if monorepo management tooling exists.

15. **dead_feature_flag_detection** (config_parsing) — Check for: stale flag reports, flag lifecycle tooling. SKIP if no feature flag system. FOUND if dead flag detection exists.

16. **heavy_dependency_detection** (config_parsing) — Check for: `bundlewatch`, `size-limit`, `webpack-bundle-analyzer` configs. Check applicability matrix for skip. FOUND if bundle size monitoring exists.

17. **progressive_rollout** (config_parsing) — Check for: canary deploy config, percentage-based rollout, blue-green deployment setup. FOUND if progressive rollout is configured.

18. **rollback_automation** (config_parsing + ci_workflow) — Check for: automated rollback on error rate, one-click rollback config, deployment rollback scripts. FOUND if automated rollback exists.

19. **version_drift_detection** (config_parsing) — Check for: `syncpack`, `manypkg`, version consistency checks in monorepo. FOUND if version drift detection exists.

### Pillar 3: Testing (8 criteria)

1. **unit_tests_exist** (file_existence) — Search for: `*_test.go`, `test_*.py`, `*_test.py`, `*.test.ts`, `*.test.js`, `*.spec.ts`, `*.spec.js`, `#[test]` in `*.rs`, `TEST_F` in `*.cpp`. FOUND if test files exist.

2. **unit_tests_runnable** (doc_content) — Check README.md, AGENTS.md, Makefile, package.json scripts for a documented test command (`go test`, `pytest`, `jest`, `cargo test`, `make test`). FOUND if test command is documented and appears runnable.

3. **integration_tests_exist** (file_existence) — Search for: Playwright/Cypress configs, directories named `acceptance/`, `integration/`, `e2e/`, files with `integration_test` or `acceptance_test` in name, `compiletest` for Rust. FOUND if integration/E2E tests exist.

4. **test_coverage_thresholds** (config_parsing) — Check for: `codecov.yml` with targets, Jest `coverageThreshold` in config, `pytest-cov` minimum in config, coverage enforcement in CI. FOUND if minimum coverage is enforced.

5. **test_naming_conventions** (code_search) — Check if test files follow language conventions: Go `*_test.go` with `TestXxx`, Python `test_*` with `test_*` functions, JS/TS `*.test.*` or `*.spec.*`. FOUND if consistent naming patterns used.

6. **test_isolation** (config_parsing) — Check for: `t.Parallel()` in Go tests, `pytest-xdist` workers, Jest `--workers`, `-race` flag in Go test commands, `cargo nextest`, test randomization flags. FOUND if test isolation is configured.

7. **flaky_test_detection** (config_parsing + ci_workflow) — Check for: `pytest-rerunfailures`, `--stress` flag, retry-on-failure config, flaky test dashboard, quarantine mechanisms. FOUND if flaky tests are detected or quarantined.

8. **test_performance_tracking** (config_parsing) — Check for: `--durations` flags in test commands, CI timing reports, `gotestsum` timing, test analytics. FOUND if test execution times are monitored.

### Pillar 4: Documentation (8 criteria)

1. **readme** (file_existence + doc_content) — Check for README.md at root. Verify it has meaningful content (setup instructions, description, usage). FOUND if comprehensive README exists.

2. **agents_md** (file_existence) — Check for: `AGENTS.md` or `CLAUDE.md` at root documenting commands, conventions, and build steps for agents. FOUND if agent instructions file exists.

3. **agents_md_validation** (ci_workflow) — Check CI workflows for jobs that validate commands documented in AGENTS.md still work. FOUND if CI validates agent instructions.

4. **documentation_freshness** (git_history) — Run `git log -1 --format="%ci" -- README.md AGENTS.md CLAUDE.md CONTRIBUTING.md 2>/dev/null` and check if any key doc was modified within 180 days. FOUND if docs are fresh.

5. **automated_doc_generation** (ci_workflow + config_parsing) — Check for: Sphinx conf.py, MkDocs config, Typedoc config, godoc generation in Makefile, `make docs` target, doc generation CI workflows. FOUND if docs are auto-generated from code.

6. **api_schema_docs** (file_existence) — Check for: `openapi.yaml`/`openapi.json`/`swagger.*`, `.proto` files, GraphQL schema files (`*.graphql`, `schema.graphql`). FOUND if API schemas are documented.

7. **service_flow_documented** (file_existence) — Check for: PlantUML `.puml` files, Mermaid diagrams in docs, `docs/architecture/` directory, architecture diagram files (`.png`, `.svg` in docs). FOUND if architecture/service flow diagrams exist.

8. **skills** (file_existence) — Check for: `.claude/skills/` or `.factory/skills/` directories with skill definitions. FOUND if reusable agent skill definitions exist.

### Pillar 5: Development Environment (5 criteria)

1. **devcontainer** (file_existence) — Check for: `.devcontainer/devcontainer.json` or `.devcontainer.json`. FOUND if devcontainer config exists.

2. **devcontainer_runnable** (config_parsing) — If devcontainer exists, check if it has a valid base image and features configured. SKIP if no devcontainer found. FOUND if devcontainer appears buildable.

3. **env_template** (file_existence) — Check for: `.env.example`, `.env.template`, `.envrc.example`, documented env vars in docs. FOUND if env template exists.

4. **local_services_setup** (file_existence) — Check for: `docker-compose.yml` or `compose.yml` with service definitions (Postgres, Redis, etc.). FOUND if local services are containerized.

5. **database_schema** (file_existence) — Check for: migration files (Alembic, Prisma, ActiveRecord, Flyway, Goose), `schema.prisma`, `schema.sql`, SQLAlchemy models. SKIP if project has no database (no migration files, no ORM, no schema files detected). FOUND if schema management exists.

### Pillar 6: Debugging & Observability (11 criteria)

1. **structured_logging** (dependency_check) — Check dependency manifests and imports for: `zap`, `logrus`, `zerolog` (Go), `structlog`, `logging` with formatters (Python), `winston`, `pino` (Node.js), `tracing` crate (Rust). FOUND if structured logging library is used.

2. **distributed_tracing** (dependency_check) — Check for: OpenTelemetry packages, `opentracing`, Jaeger client, `X-Request-ID` middleware, `dd-trace`. FOUND if distributed tracing is instrumented.

3. **metrics_collection** (dependency_check) — Check for: Prometheus client libraries, Datadog StatsD, OpenTelemetry metrics, custom metrics packages. FOUND if metrics are collected.

4. **alerting_configured** (config_parsing) — Check for: Prometheus Alertmanager configs, PagerDuty integration, OpsGenie config, alert rules files. FOUND if alerting is configured.

5. **deployment_observability** (config_parsing + doc_content) — Check for: Grafana dashboard configs, monitoring documentation, deploy notification configs, dashboard links in docs. FOUND if deployment monitoring exists.

6. **error_tracking_contextualized** (dependency_check) — Check for: Sentry SDK with release tracking, Bugsnag, Rollbar, error tracking with context in dependencies. FOUND if error tracking service is configured.

7. **health_checks** (code_search) — Search for: `/health` or `/healthz` endpoint definitions, liveness/readiness probe configs, health check functions. FOUND if health check endpoints exist.

8. **profiling_instrumentation** (code_search + dependency_check) — Check for: `pprof` imports (Go), `py-spy`/`Pyroscope` (Python), profiling middleware, `Performance.md`, tracy (Rust/C++). FOUND if profiling infrastructure exists.

9. **code_quality_metrics** (ci_workflow) — Check for: Codecov integration, CodeQL workflows, SonarQube/SonarCloud config, CodeClimate, coverage upload in CI. FOUND if code quality is tracked via CI.

10. **circuit_breakers** (dependency_check) — Check for: circuit breaker libraries (`sony/gobreaker`, `resilience4j`, `hystrix`, `cockatiel`), retry-with-backoff patterns. FOUND if resilience patterns are implemented.

11. **runbooks_documented** (file_existence) — Check for: `runbooks/` directory, `INCIDENT_RESPONSE.md`, `PLAYBOOK.md`, operational docs with incident procedures. FOUND if runbooks exist.

### Pillar 7: Security & Governance (11 criteria)

1. **branch_protection** (api_check) — Run `gh api repos/{owner}/{repo}/rulesets 2>/dev/null` or `gh api repos/{owner}/{repo}/branches/main/protection 2>/dev/null`. SKIP if gh unavailable. FOUND if branch protection rules exist.

2. **codeowners** (file_existence) — Check for: `.github/CODEOWNERS`, `CODEOWNERS`, `docs/CODEOWNERS`. FOUND if CODEOWNERS file exists with team assignments.

3. **secret_scanning** (api_check) — Run `gh api repos/{owner}/{repo} --jq '.security_and_analysis.secret_scanning.status' 2>/dev/null`. SKIP if gh unavailable or insufficient permissions. FOUND if secret scanning is enabled.

4. **secrets_management** (config_parsing + code_search) — Check for: references to vault (HashiCorp Vault, AWS Secrets Manager), `secrets.*` patterns in config, GitHub Actions secrets usage, SOPS config, `.envrc` gitignored. FOUND if secrets use a vault or manager.

5. **dependency_update_automation** (file_existence) — Check for: `.github/dependabot.yml`, `renovate.json`, `renovate.json5`, `.renovaterc`. FOUND if dependency update bot is configured.

6. **gitignore_comprehensive** (config_parsing) — Read `.gitignore` and check if it covers: `.env*`, credentials, `node_modules/` or equivalent, build artifacts, IDE configs (`.idea/`, `.vscode/`). FOUND if gitignore properly excludes sensitive files and artifacts.

7. **automated_security_review** (ci_workflow) — Check for: CodeQL analysis workflow, Snyk, Trivy, Semgrep, SAST pipeline steps in CI. FOUND if security scanning runs in CI.

8. **log_scrubbing** (code_search) — Search for: redaction functions, `SafeValue` interfaces, password masking patterns, sanitization middleware, log field filtering. FOUND if sensitive data is redacted from logs.

9. **pii_handling** (code_search) — Search for: data classification annotations, GDPR controls, redactability systems, PII detection tooling. SKIP if project doesn't handle user data. FOUND if PII is classified and handled.

10. **dast_scanning** (ci_workflow) — Check for: OWASP ZAP, Burp Suite, Nuclei in CI workflows, dynamic scanning steps. Check applicability matrix for skip. FOUND if DAST runs in CI.

11. **privacy_compliance** (config_parsing) — Check for: data retention policies, consent management, privacy tooling, GDPR/CCPA controls. FOUND if privacy compliance is enforced.

### Pillar 8: Task Discovery (4 criteria)

1. **issue_templates** (file_existence) — Check for: `.github/ISSUE_TEMPLATE/` directory with template files, `.github/ISSUE_TEMPLATE.md`. FOUND if structured issue templates exist.

2. **issue_labeling_system** (api_check) — Run `gh label list --limit 50 2>/dev/null` and check for organized label taxonomy (priority, type, area labels). SKIP if gh unavailable. FOUND if comprehensive labels exist.

3. **backlog_health** (api_check) — Run `gh issue list --limit 20 --json title,labels 2>/dev/null`. Check if >70% of issues have labels and titles >10 chars. SKIP if gh unavailable. FOUND if backlog is well-maintained.

4. **pr_templates** (file_existence) — Check for: `.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE/`, `PULL_REQUEST_TEMPLATE.md`. FOUND if PR template exists with structured sections.

### Pillar 9: Product & Analytics (3 criteria)

1. **product_analytics_instrumentation** (dependency_check) — Check for: Mixpanel, Amplitude, PostHog, Heap, GA4, custom analytics SDKs in dependencies. Check applicability matrix for skip. FOUND if analytics is instrumented.

2. **error_to_insight_pipeline** (config_parsing) — Check for: Sentry-GitHub integration, automated issue creation from errors, error tracking to ticket pipeline. FOUND if errors automatically create trackable issues.

3. **experiment_infrastructure** (config_parsing) — Check for: A/B testing framework, feature flags with metrics integration, experiment platform config. FOUND if experiment infrastructure exists.

### Custom Criteria (from .agentfit.yml)

If `.agentfit.yml` was found and contains `custom_criteria`, evaluate each valid custom criterion:
- Use the `check` field to determine what to look for in the codebase
- Use the `found_when` field to determine the condition for FOUND status
- Assign to the specified `pillar` and `level`
- Record status, score, and evidence just like default criteria
- Mark custom criteria with a `custom: true` flag for report rendering

If `.agentfit.yml` contains `disabled_criteria`, remove those criteria from the assessment entirely before scoring. They should not appear in the report at all.

## Step 3: Calculate Scores

### Per-Pillar Scores

For each pillar, calculate:
- `criteria_passed` = count of criteria with status `found`
- `criteria_total` = count of criteria with status `found` or `missing` (exclude `skipped`)
- `percentage` = round(`criteria_passed` / `criteria_total` * 100)
- Display as: `criteria_passed`/`criteria_total` (`percentage`%)

### Overall Pass Rate

- `total_passed` = sum of all `criteria_passed` across all pillars
- `total_applicable` = sum of all `criteria_total` across all pillars
- `pass_rate` = round(`total_passed` / `total_applicable` * 100)

### Weighted Score

Each criterion has an impact tier that determines its weight in the weighted score calculation:
- **High (3x)**: type_check, lint_config, unit_tests_exist, agents_md, readme, deps_pinned, secrets_management, gitignore_comprehensive, pre_commit_hooks, single_command_setup
- **Medium (2x)**: All criteria not listed as high or low
- **Low (1x)**: duplicate_code_detection, tech_debt_tracking, code_modularization, naming_consistency, build_performance_tracking, dead_feature_flag_detection, heavy_dependency_detection, privacy_compliance

Custom criteria from `.agentfit.yml` use their specified `impact_tier` (default: medium).

Calculate: `weighted_score` = round(`sum(weight * passed)` / `sum(weight * applicable)` * 100)

Where `passed` = 1 if found, 0 if missing. Skipped criteria are excluded from both numerator and denominator.

### Maturity Level Calculation

Each criterion is assigned to a maturity level (L1-L5). Calculate completion percentage for each level using only the criteria assigned to that level.

**Level-to-criteria mapping:**

**Level 1 (Functional):** formatter, lint_config, type_check, strict_typing, unit_tests_exist, unit_tests_runnable, test_naming_conventions, readme, documentation_freshness, build_cmd_doc, deps_pinned, vcs_cli_tools, gitignore_comprehensive

**Level 2 (Documented):** pre_commit_hooks, naming_consistency, agents_md, skills, devcontainer, env_template, local_services_setup, database_schema, single_command_setup, branch_protection, codeowners, secrets_management, issue_templates, pr_templates, structured_logging

**Level 3 (Standardized):** large_file_detection, code_modularization, cyclomatic_complexity, dead_code_detection, duplicate_code_detection, tech_debt_tracking, integration_tests_exist, test_coverage_thresholds, test_isolation, agents_md_validation, automated_doc_generation, api_schema_docs, service_flow_documented, distributed_tracing, metrics_collection, health_checks, code_quality_metrics, secret_scanning, dependency_update_automation, automated_security_review, log_scrubbing, issue_labeling_system, backlog_health, fast_ci_feedback, release_automation, release_notes_automation, unused_deps_detection

**Level 4 (Optimized):** n_plus_one_detection, deployment_frequency, automated_pr_review, agentic_development, feature_flag_infrastructure, build_performance_tracking, monorepo_tooling, flaky_test_detection, test_performance_tracking, devcontainer_runnable, alerting_configured, deployment_observability, error_tracking_contextualized, profiling_instrumentation, circuit_breakers, runbooks_documented, pii_handling

**Level 5 (Autonomous):** dead_feature_flag_detection, heavy_dependency_detection, progressive_rollout, rollback_automation, version_drift_detection, product_analytics_instrumentation, error_to_insight_pipeline, experiment_infrastructure, dast_scanning, privacy_compliance

**Gated progression rule:** To unlock level N, the repository must pass ≥80% of applicable criteria at level N AND all previous levels. Calculate each level's completion percentage (excluding skipped criteria). The maturity level is the highest level where the 80% gate is met for that level and all levels below it.

### Strengths and Opportunities

**Strengths (top 3):** Select the 3 pillars with the highest percentage scores. For each, list the pillar name with percentage and 2-3 key passing criteria as evidence.

**Opportunities (top 3):** Select the 3 most impactful MISSING criteria, prioritized by maturity level (L1 gaps first, then L2, then L3, etc.). For each, provide the criterion name and a specific remediation action.

### Summary Headline

Generate a short headline based on the strongest pillar or most notable pattern. Examples:
- "Strong Testing" (if Testing pillar is highest)
- "Well-Documented" (if Documentation pillar is highest)
- "Security-First" (if Security pillar is highest)
- "Needs Foundation" (if Level 1 criteria are failing)

Then write a 1-2 sentence summary: "{project_name} reaches Level {N} with {pass_rate}% pass rate. Currently reaching {maturity_label} grade with {total_passed}/{total_applicable} criteria passing ({pass_rate}%). Key areas for improvement include the opportunities listed below."

## Step 4: Generate HTML Report

Create an HTML file with the complete assessment results. Write it to a temporary file and open it in the browser.

Use the Bash tool to write the HTML file. The file should be written to `/tmp/agentfit-report-{project_name}.html`.

Also write a JSON sidecar file to `/tmp/agentfit-report-{project_name}-{git_sha}.json` with this structure:
```json
{
  "schema_version": "{skill_version}",
  "metadata": {
    "assessment_date": "{ISO 8601 datetime}",
    "git_sha": "{git_sha}",
    "skill_version": "{skill_version}",
    "project_types": ["{type1}", "{type2}"],
    "total_criteria": {N},
    "skipped_criteria": {N},
    "custom_criteria_count": {N}
  },
  "scores": {
    "pass_rate": {pass_rate},
    "weighted_score": {weighted_score},
    "maturity_level": {N}
  },
  "pillars": {
    "{pillar_name}": { "passed": {N}, "total": {N}, "percentage": {N} }
  },
  "criteria": {
    "{criterion_name}": { "status": "found|missing|skipped", "evidence": "...", "pillar": "...", "level": "L{N}", "impact_tier": "high|medium|low", "custom": false }
  }
}
```

After writing both files, open the HTML with: `open /tmp/agentfit-report-{project_name}.html 2>/dev/null || xdg-open /tmp/agentfit-report-{project_name}.html 2>/dev/null || echo "Report saved to /tmp/agentfit-report-{project_name}.html"`

### HTML Template

The HTML report MUST use this structure with inline CSS. Replace all `{placeholder}` values with actual assessment data:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{project_name} — Agent Fit Report</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    background: #0d0d1a;
    color: #e0e0e0;
    padding: 40px 60px;
    line-height: 1.6;
  }
  h1 {
    font-size: 3rem;
    font-weight: 300;
    margin-bottom: 4px;
    color: #ffffff;
  }
  .badge {
    display: inline-block;
    font-size: 0.8rem;
    font-weight: 600;
    padding: 2px 10px;
    border: 1px solid #888;
    border-radius: 12px;
    vertical-align: middle;
    margin-left: 12px;
    color: #ccc;
    text-transform: uppercase;
  }
  .meta {
    font-family: 'SF Mono', 'Fira Code', 'Consolas', monospace;
    font-size: 0.85rem;
    color: #888;
    margin-bottom: 4px;
  }
  .description {
    font-size: 0.9rem;
    color: #666;
    margin-bottom: 24px;
  }

  /* Level Progress Bar */
  .level-bar {
    display: flex;
    height: 8px;
    border-radius: 4px;
    overflow: hidden;
    margin-bottom: 8px;
    background: #1a1a2e;
  }
  .level-segment {
    height: 100%;
    position: relative;
  }
  .level-labels {
    display: flex;
    margin-bottom: 40px;
    font-size: 0.75rem;
    font-family: monospace;
  }
  .level-label {
    display: flex;
    align-items: center;
    gap: 4px;
  }
  .level-label .pct { color: #ccc; }
  .level-label .name { color: #888; }

  .l1 { background: #2d8a4e; }
  .l2 { background: #2d8a4e; }
  .l3 { background: #2d8a4e; }
  .l4 { background: #d4a017; }
  .l5 { background: #333; }

  /* Summary */
  .summary-section { margin-bottom: 48px; }
  .summary-section h2 {
    font-size: 1.8rem;
    font-weight: 600;
    color: #ffffff;
    margin-bottom: 16px;
    margin-top: 40px;
  }
  .summary-section p {
    font-family: monospace;
    font-size: 0.85rem;
    color: #999;
    max-width: 700px;
    line-height: 1.7;
    margin-bottom: 32px;
  }

  /* Strengths / Opportunities */
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 60px;
  }
  .col-header {
    font-size: 0.7rem;
    letter-spacing: 2px;
    text-transform: uppercase;
    color: #666;
    margin-bottom: 20px;
  }
  .highlight {
    margin-bottom: 20px;
  }
  .highlight-num {
    font-size: 0.85rem;
    font-weight: 700;
    margin-bottom: 4px;
  }
  .highlight-num.green { color: #e85d3a; }
  .highlight-num.orange { color: #d4a017; }
  .highlight-title {
    font-size: 1rem;
    font-weight: 600;
    color: #ffffff;
    margin-bottom: 4px;
  }
  .highlight-detail {
    font-size: 0.8rem;
    color: #888;
    line-height: 1.5;
  }

  /* Criteria Section */
  .criteria-section {
    margin-top: 48px;
    border-top: 1px solid #1a1a2e;
    padding-top: 32px;
  }
  .criteria-header {
    font-size: 0.7rem;
    letter-spacing: 2px;
    text-transform: uppercase;
    color: #666;
    margin-bottom: 24px;
  }
  .pillar-group { margin-bottom: 32px; }
  .pillar-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 0;
    border-bottom: 1px solid #1a1a2e;
    margin-bottom: 8px;
  }
  .pillar-name {
    font-size: 0.95rem;
    font-weight: 600;
    color: #ffffff;
  }
  .pillar-score {
    font-size: 0.85rem;
    color: #888;
    font-family: monospace;
  }
  .criterion-row {
    display: grid;
    grid-template-columns: 24px 200px 50px 1fr;
    gap: 16px;
    padding: 6px 0;
    align-items: baseline;
    font-size: 0.8rem;
  }
  .status-icon { text-align: center; }
  .status-icon.pass { color: #4caf50; }
  .status-icon.fail { color: #e85d3a; }
  .status-icon.skip { color: #555; }
  .criterion-name {
    font-family: monospace;
    color: #ccc;
  }
  .criterion-score {
    font-family: monospace;
    color: #888;
    text-align: center;
  }
  .criterion-evidence {
    color: #777;
    line-height: 1.4;
  }
</style>
</head>
<body>

<!-- HEADER -->
<h1>{project_name} <span class="badge">{language}</span><!-- Repeat for each detected project type --><span class="badge">{project_type}</span></h1>
<p class="meta">{repo_path}&nbsp;&nbsp;&nbsp;PASS RATE {pass_rate}%&nbsp;&nbsp;&nbsp;WEIGHTED {weighted_score}%</p>
<p class="description">{project_description}</p>

<!-- LEVEL PROGRESS BAR -->
<div class="level-bar">
  <div class="level-segment l1" style="width:20%"></div>
  <div class="level-segment l2" style="width:20%"></div>
  <div class="level-segment l3" style="width:20%"></div>
  <div class="level-segment l4" style="width:20%"></div>
  <div class="level-segment l5" style="width:20%"></div>
</div>
<div class="level-labels">
  <div class="level-label" style="width:20%"><span class="pct">{l1_pct}%</span>&nbsp;<span class="name">L1</span></div>
  <div class="level-label" style="width:20%"><span class="pct">{l2_pct}%</span>&nbsp;<span class="name">L2</span></div>
  <div class="level-label" style="width:20%"><span class="pct">{l3_pct}%</span>&nbsp;<span class="name">L3</span></div>
  <div class="level-label" style="width:20%"><span class="pct">{l4_pct}%</span>&nbsp;<span class="name">L4</span></div>
  <div class="level-label" style="width:20%"><span class="pct">{l5_pct}%</span>&nbsp;<span class="name">L5</span></div>
</div>

<!-- SUMMARY -->
<div class="summary-section">
  <h2>{summary_headline}</h2>
  <p>{summary_text}</p>

  <div class="columns">
    <div>
      <div class="col-header">STRENGTHS</div>
      <!-- Repeat for each strength (top 3) -->
      <div class="highlight">
        <div class="highlight-num green">01</div>
        <div class="highlight-title">{strength_1_title}</div>
        <div class="highlight-detail">{strength_1_detail}</div>
      </div>
      <!-- 02, 03 ... -->
    </div>
    <div>
      <div class="col-header">OPPORTUNITIES</div>
      <!-- Repeat for each opportunity (top 3) -->
      <div class="highlight">
        <div class="highlight-num orange">01</div>
        <div class="highlight-title">{opportunity_1_title}</div>
        <div class="highlight-detail">{opportunity_1_detail}</div>
      </div>
      <!-- 02, 03 ... -->
    </div>
  </div>
</div>

<!-- ALL CRITERIA -->
<div class="criteria-section">
  <div class="criteria-header">ALL CRITERIA</div>

  <!-- Repeat for each pillar (9 total) -->
  <div class="pillar-group">
    <div class="pillar-header">
      <span class="pillar-name">{pillar_name}</span>
      <span class="pillar-score">{passed}/{total} ({percentage}%)</span>
    </div>

    <!-- Repeat for each criterion in this pillar, alphabetically by name -->
    <div class="criterion-row">
      <span class="status-icon {pass|fail|skip}">{✓|✗|—}</span>
      <span class="criterion-name">{criterion_name}</span>
      <span class="criterion-score">{1/1|0/1|—/—}</span>
      <span class="criterion-evidence">{evidence_text}</span>
    </div>
  </div>
</div>

<!-- FOOTER -->
<footer style="margin-top: 64px; padding-top: 24px; border-top: 1px solid #1a1a2e; font-size: 0.75rem; color: #555; font-family: monospace;">
  Agent-Fit v{skill_version} &middot; Assessed {assessment_date} &middot; {git_sha} &middot; {total_criteria} criteria evaluated &middot; {skipped_criteria} skipped &middot; {project_types}
</footer>

</body>
</html>
```

**IMPORTANT**: When generating the HTML, replace ALL `{placeholder}` values with actual data from the assessment. Do NOT leave any placeholders in the output. Generate the complete HTML string and write it to the temp file in a single Bash command.

## Step 5: Report to User

After opening the HTML report, print a brief summary to the console:

```
Agent Fit Report: {project_name}
Type: {project_types} | Language: {language} | v{skill_version}
Pass Rate: {pass_rate}% | Weighted: {weighted_score}% | Level: L{maturity_level} ({maturity_label})
Report: /tmp/agentfit-report-{project_name}.html
JSON:   /tmp/agentfit-report-{project_name}-{git_sha}.json

Pillars:
  Style & Validation:       {p1_passed}/{p1_total} ({p1_pct}%)
  Build System:             {p2_passed}/{p2_total} ({p2_pct}%)
  Testing:                  {p3_passed}/{p3_total} ({p3_pct}%)
  Documentation:            {p4_passed}/{p4_total} ({p4_pct}%)
  Dev Environment:          {p5_passed}/{p5_total} ({p5_pct}%)
  Debugging & Observability:{p6_passed}/{p6_total} ({p6_pct}%)
  Security & Governance:    {p7_passed}/{p7_total} ({p7_pct}%)
  Task Discovery:           {p8_passed}/{p8_total} ({p8_pct}%)
  Product & Analytics:      {p9_passed}/{p9_total} ({p9_pct}%)
```
