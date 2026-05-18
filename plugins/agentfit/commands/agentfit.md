---
description: Evaluates codebase agent-readiness across 9 pillars, 80+ criteria, and 5 maturity levels — produces an HTML report with pass rate scoring
---

When invoked, perform the following assessment. This is a READ-ONLY analysis — do NOT modify any files in the target codebase. The only file you create is the HTML report.

## Step 1: Discover Project Context

1. List root directory files and folders.
2. Detect ALL languages present by checking for these manifest files:
   - `go.mod` → Go
   - `Cargo.toml` → Rust
   - `pyproject.toml` or `setup.py` or `requirements.txt` → Python
   - `pom.xml` or `build.gradle` or `build.gradle.kts` → Java/Kotlin
   - `package.json` with `tsconfig.json` → TypeScript
   - `package.json` without `tsconfig.json` → JavaScript
   - `CMakeLists.txt` or `Makefile` with `.cpp`/`.cc`/`.h` files → C++
   - `Package.swift` → Swift
   - `mix.exs` → Elixir
   - If none found → Unknown
   The **primary language** is the one with the most source files. Secondary languages are listed in the badge as `Primary + Secondary` (e.g., `Go + TypeScript`). Evaluate language-specific criteria for ALL detected languages — a criterion passes if it is satisfied for ANY detected language.
3. Extract project name: use the current directory name.
4. Extract repo path: run `git remote get-url origin 2>/dev/null` and parse `org/repo` from the URL. If no git remote, use the directory path.
5. Get a short project description from README.md first line/paragraph if available.

### Grounding on Previous Reports

If a previous report exists at `/tmp/agentfit-report-{project_name}.json`, load it and use its criterion statuses as a reference baseline. When evaluating each criterion:
- If the previous status was `found` and the current evidence still supports it, maintain `found`
- If the previous status was `missing` and no new evidence is found, maintain `missing`
- Only change a criterion's status if the underlying signal has demonstrably changed (e.g., a config file was added or removed)
- In the evidence field, note when a status changed from the previous report: "Changed from missing → found: [reason]"

If no previous report exists, evaluate all criteria fresh.

### Custom Criteria Configuration

Check for `.agentfit.yml` at the repository root. If found, parse it:

- **`custom_criteria`**: Array of additional criteria to evaluate alongside built-ins. Each entry has:
  - `name` (snake_case identifier), `pillar` (1-9), `level` (1-5), `check` (what to look for), `found_when` (pass condition), `impact` (`high`|`medium`|`low`, default: `medium`)
  - Evaluate each custom criterion using the same signal types as built-in criteria. Include results in the matching pillar's output.
- **`disabled_criteria`**: Array of built-in criterion names to force-skip. Mark each as `—/—` with evidence "Disabled via .agentfit.yml".

If no `.agentfit.yml` exists, skip this step.

## Step 2: Evaluate All Criteria

Evaluate each criterion below using the specified signal type. For each criterion, record:
- **status**: `found` (passes), `missing` (fails), or `skipped` (not applicable)
- **score**: `1/1`, `0/1`, or `—/—`
- **evidence**: A specific sentence referencing the actual file/config found, or what is missing, or why it was skipped. For file-existence criteria, include counts where meaningful (e.g., "Found 847 test files (*_test.go) across 12 packages" not just "Found test files"). For threshold-based criteria, include the measured value and threshold.

### Project Type Detection

Before evaluating criteria, detect the project type to determine which skip rules apply:

- **Monorepo**: Check for `pnpm-workspace.yaml`, `lerna.json`, `nx.json`, `turbo.json`, Cargo workspace members in `Cargo.toml`, or multiple `go.mod` files. If any found, mark as monorepo.
- **Library**: Check if `package.json` has no `bin` field and has a `main`/`module`/`exports` field, or if the project is a Go module without `cmd/`, or a Rust crate with `[lib]` in `Cargo.toml` and no `[[bin]]`. If so, mark as library.
- **CLI tool**: Check if `package.json` has a `bin` field, or `cmd/` directory exists (Go), or `[[bin]]` in `Cargo.toml`, or `entry_points.console_scripts` in Python config.
- **Web app**: Check for frontend frameworks (Next.js, React, Vue, Angular, Svelte) in dependencies, or `index.html`, or web server routes.
- **API service**: Check for HTTP server frameworks (Express, Gin, FastAPI, Actix, Flask) without frontend assets.

Record the detected project type. A project may match multiple types (e.g., monorepo + web app).

### Criteria Skip Rules

Skip a criterion (mark as `—/—`) when:
- It requires `gh` CLI but `gh auth status` fails or `gh` is not installed
- It checks for database tooling but the project has no database
- It checks for deployment/server tooling but the project is a library or CLI tool
- It checks for N+1 queries but the project doesn't use an ORM
- It checks for PII/privacy but the project doesn't handle user data
- It checks for DAST but the project is a CLI tool or library
- It checks for monorepo tooling (`monorepo_tooling`, `version_drift_detection`) but the project is not a monorepo
- It checks for devcontainer runnability (`devcontainer_runnable`) but no devcontainer config exists
- It checks for dead feature flags (`dead_feature_flag_detection`) but no feature flag system is detected
- It checks for product analytics (`product_analytics_instrumentation`) but the project is a library, CLI tool, or server infrastructure
- It checks for bundle size (`heavy_dependency_detection`) but the project is not a frontend/bundled project

When skipping, always explain why in the evidence field.

### Pillar 1: Style & Validation (13 criteria)

Evaluate each criterion. For all config-parsing criteria, read the actual config file and report what rules/settings are configured.

1. **formatter** (config_parsing) — Check for: `.prettierrc*`, `biome.json`, `pyproject.toml` with `[tool.black]` or `[tool.ruff.format]`, `rustfmt.toml`, `.clang-format`, `.editorconfig` with formatting rules, `gofumpt`/`goimports` in Makefile or CI, `google-java-format` or `spotless` in Gradle/Maven config, `ktlint` config, `mix format` config (Elixir). FOUND if any formatter is configured.

2. **lint_config** (config_parsing) — Check for: `.eslintrc*`, `eslint.config.*`, `biome.json` with linter, `.golangci.yml`, `pyproject.toml` with `[tool.ruff]` or `[tool.pylint]`, `clippy.toml` or `Cargo.toml` with clippy config, `.clang-tidy`, Checkstyle config (`checkstyle.xml`), PMD ruleset (`pmd.xml`), SpotBugs/FindBugs config, `detekt.yml` (Kotlin), `credo` config (Elixir). FOUND if linter configured with specific rules.

3. **type_check** (config_parsing) — Check for: `tsconfig.json`, `mypy.ini` or `pyproject.toml` with `[tool.mypy]`, Go compiler (always passes for Go), Rust compiler (always passes for Rust), Java compiler (always passes for Java/Kotlin), `pyrightconfig.json`. FOUND if type checking is configured.

4. **strict_typing** (config_parsing) — Check for: `tsconfig.json` with `"strict": true`, `mypy` with `strict = true` or `disallow_untyped_defs = true`, Go (always strict), Rust (always strict), Java (always strict), Kotlin (always strict), Clippy pedantic. FOUND if strict mode is enabled.

5. **pre_commit_hooks** (file_existence) — Check for: `.pre-commit-config.yaml`, `.husky/` directory, `.lefthook.yml`, `lint-staged` in `package.json`, `scripts/pre-commit*`. FOUND if any pre-commit hook framework is configured.

6. **naming_consistency** (config_parsing + doc_content) — Check for: ESLint `naming-convention` rule, `revive` linter naming rules in golangci-lint, documented naming conventions in AGENTS.md/CLAUDE.md/CONTRIBUTING.md, `.editorconfig`. FOUND if naming conventions are documented or enforced via linter.

7. **large_file_detection** (file_existence) — Check for: `.gitattributes` with LFS entries, `pre-commit` hook `check-added-large-files`, CI jobs checking file size. FOUND if large file detection is configured.

8. **code_modularization** (config_parsing) — Check for: Go `internal/` packages, `eslint-plugin-boundaries`, Nx boundaries config, Rust module visibility, monorepo workspace boundaries. FOUND if module boundary enforcement exists.

9. **cyclomatic_complexity** (config_parsing) — Check for: `gocyclo` or `cyclop` in golangci-lint config, ESLint `complexity` rule, Ruff `C901`, Clippy `cognitive_complexity`. FOUND if complexity analysis tools are configured.

10. **dead_code_detection** (config_parsing) — Check for: `knip` in package.json (knip code analysis mode — detects unused exports, files, and types), `vulture` in Python config, `staticcheck` unused checks, Ruff `F401`/`F841`, Clippy `dead_code` warnings, `deadcode` tool. FOUND if dead code detection is configured.

11. **duplicate_code_detection** (config_parsing) — Check for: `jscpd` config or in CI, `PMD CPD`, `Simian`, golangci-lint `dupl` linter, SonarQube duplication. FOUND if duplication scanner is configured.

12. **tech_debt_tracking** (ci_workflow) — Check for: CI workflows scanning for TODO/FIXME markers, `godox` linter in golangci-lint, SonarQube, `check-todos` scripts. FOUND if tech debt tracking is automated.

13. **n_plus_one_detection** (config_parsing) — Check for: `bullet` gem (Ruby), `django-auto-prefetch` or `nplusone` (Python), Prisma query warnings, ORM query analyzers. SKIP if project doesn't use an ORM or database. FOUND if N+1 detection is configured.

### Pillar 2: Build System (19 criteria)

1. **build_cmd_doc** (doc_content) — Check README.md, AGENTS.md, CLAUDE.md, Makefile, Taskfile.yml, Justfile, Earthfile, CONTRIBUTING.md for a code block or command documenting build/install/run instructions. FOUND if at least one of these files contains a code block or target with build/run/install instructions. Evidence must cite which file and what command was found.

2. **deps_pinned** (file_existence) — Check for: `go.sum`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `uv.lock`, `poetry.lock`, `Cargo.lock`, `Pipfile.lock`, `gradle.lockfile` or `buildSrc/*.lock`, `mix.lock` (Elixir). FOUND if lockfile exists. Note: Rust libraries intentionally gitignore Cargo.lock — skip for Rust library projects. Note: Maven/Gradle projects often pin versions in POM/build files directly — check for version pinning in `pom.xml` or `build.gradle` if no lockfile.

3. **vcs_cli_tools** (doc_content + file_existence) — Check for: VCS workflow tooling documented in Makefile, CONTRIBUTING.md, or README.md (e.g., PR creation commands, branch policies), OR `.github/workflows/*.yml` referencing `gh` CLI commands. FOUND if VCS tooling usage is documented or automated in CI.

4. **single_command_setup** (doc_content) — Check README.md, AGENTS.md, CONTRIBUTING.md for a code block with a single setup command (e.g., `make setup`, `task setup`, `just setup`, `earthly +setup`, `docker-compose up`, `npm install`, `nix develop`, `./dev doctor`) that completes project setup (not a multi-step procedure). FOUND if a single setup command is documented in a code block. Evidence must quote the command found.

5. **fast_ci_feedback** (ci_workflow) — Read `.github/workflows/*.yml`. FOUND if CI workflow uses caching (actions/cache, turbo, sccache, or language-specific caching) AND does not include matrix > 5 targets without parallelization. Evidence must cite specific caching actions and matrix size found.

6. **deployment_frequency** (git_history) — Run `git tag --sort=-creatordate | head -20` and check dates. FOUND if multiple releases per month or regular release cadence visible.

7. **release_automation** (ci_workflow) — Check for: `.github/workflows/release*`, `goreleaser.yml`, `.releaserc` or `release.config.*` (semantic-release), `release-please-config.json` (release-please), `cargo-release` config, publish workflows, tag-triggered release workflows. FOUND if releases are automated via CI.

8. **release_notes_automation** (config_parsing) — Check for: `.goreleaser.yml` with changelog, `towncrier.toml`, `.changeset/`, `git-cliff.toml`, `cliff.toml`, release-changelog-builder action in CI, `release-please-config.json`, `.releaserc` with changelog plugin. FOUND if release notes are auto-generated.

9. **automated_pr_review** (ci_workflow + code_search) — Check for: CodeRabbit, Copilot Code Review, Danger.js, custom review bots in CI, Semgrep in PR checks, CodeAnt AI, Graphite Reviewer, Sourcery, `claude-code` review in CI. FOUND if automated review comments are generated on PRs.

10. **agentic_development** (git_history) — Run `git log --format="%aN" -100 2>/dev/null` and check for agent co-author signatures (Claude, Copilot, factory-droid, Devin, Sweep). FOUND if agent co-authorship is visible in recent git history. Note: file-existence checks for AGENTS.md and .claude/skills/ are already covered by the `agents_md` and `skills` criteria in Pillar 4 — this criterion only checks the unique signal of agents actively contributing code.

11. **feature_flag_infrastructure** (config_parsing + dependency_check) — Check for: LaunchDarkly, Statsig, Unleash, custom feature flag config files, `crates/feature_flags`. FOUND if feature flag system is configured.

12. **build_performance_tracking** (config_parsing) — Check for: Bazel distributed cache config, Turborepo remote cache (`turbo.json` with `remoteCache`), Nx Cloud (`nx.json` with `nxCloudAccessToken` or `nx-cloud.env`), sccache, Gradle build scans (`--scan` flag or `com.gradle.enterprise` plugin), build metrics export, CI build timing with `actions/cache`. FOUND if build times are cached or tracked.

13. **unused_deps_detection** (ci_workflow + config_parsing) — Check for: `depcheck` in scripts, `go mod tidy` in CI, `cargo-udeps`, `deptry`, `knip` in package.json (knip dependency mode — detects unused packages in node_modules). FOUND if unused deps are detected automatically.

14. **monorepo_tooling** (config_parsing) — Check for: Lerna, Nx (`nx.json`), Turborepo (`turbo.json`), Bazel BUILD files, Cargo workspace members, `pnpm-workspace.yaml`, Gradle multi-project builds, Earthly multi-target builds. SKIP if not a monorepo. FOUND if monorepo management tooling exists.

15. **dead_feature_flag_detection** (config_parsing) — Check for: stale flag reports, flag lifecycle tooling. SKIP if no feature flag system. FOUND if dead flag detection exists.

16. **heavy_dependency_detection** (config_parsing) — Check for: `bundlewatch`, `size-limit`, `webpack-bundle-analyzer` configs. SKIP if not a frontend/bundled project. FOUND if bundle size monitoring exists.

17. **progressive_rollout** (config_parsing) — Check for: canary deploy config, percentage-based rollout, blue-green deployment setup. FOUND if progressive rollout is configured.

18. **rollback_automation** (config_parsing + ci_workflow) — Check for: automated rollback on error rate, one-click rollback config, deployment rollback scripts. FOUND if automated rollback exists.

19. **version_drift_detection** (config_parsing) — Check for: `syncpack`, `manypkg`, version consistency checks in monorepo. SKIP if not a monorepo. FOUND if version drift detection exists.

### Pillar 3: Testing (8 criteria)

1. **unit_tests_exist** (file_existence) — Search for: `*_test.go`, `test_*.py`, `*_test.py`, `*.test.ts`, `*.test.js`, `*.spec.ts`, `*.spec.js`, `#[test]` in `*.rs`, `TEST_F` in `*.cpp`, `src/test/` directory (Java/Kotlin), `*Test.java`, `*Spec.kt`, `*_test.exs` (Elixir). FOUND if test files exist.

2. **unit_tests_runnable** (doc_content) — Check README.md, AGENTS.md, Makefile, Taskfile.yml, Justfile, package.json scripts for a documented test command (`go test`, `pytest`, `jest`, `cargo test`, `make test`, `./gradlew test`, `mvn test`, `mix test`). FOUND if test command is documented and appears runnable.

3. **integration_tests_exist** (file_existence) — Search for: Playwright/Cypress configs, directories named `acceptance/`, `integration/`, `e2e/`, files with `integration_test` or `acceptance_test` in name, `compiletest` for Rust. FOUND if integration/E2E tests exist.

4. **test_coverage_thresholds** (config_parsing) — Check for: `codecov.yml` with targets, Jest `coverageThreshold` in config, `pytest-cov` minimum in config, JaCoCo minimum coverage rules in Gradle/Maven, coverage enforcement in CI. FOUND if minimum coverage is enforced.

5. **test_naming_conventions** (code_search) — Check if test files follow language conventions: Go `*_test.go` with `TestXxx`, Python `test_*` with `test_*` functions, JS/TS `*.test.*` or `*.spec.*`, Java/Kotlin `*Test.java`/`*Spec.kt` in `src/test/`, Rust `#[test]` in `mod tests`, Elixir `*_test.exs` in `test/`. FOUND if consistent naming patterns used.

6. **test_isolation** (config_parsing) — Check for: `t.Parallel()` in Go tests, `pytest-xdist` workers, Jest `--workers`, `-race` flag in Go test commands, `cargo nextest`, test randomization flags, JUnit `@Execution(CONCURRENT)`, Gradle `maxParallelForks`, Maven Surefire `forkCount`. FOUND if test isolation is configured.

7. **flaky_test_detection** (config_parsing + ci_workflow) — Check for: `pytest-rerunfailures`, `--stress` flag, retry-on-failure config, flaky test dashboard, quarantine mechanisms. FOUND if flaky tests are detected or quarantined.

8. **test_performance_tracking** (config_parsing) — Check for: `--durations` flags in test commands, CI timing reports, `gotestsum` timing, test analytics. FOUND if test execution times are monitored.

### Pillar 4: Documentation (10 criteria)

1. **readme** (file_existence + doc_content) — Check for README.md at root. FOUND if README contains at least 2 of: project description, installation instructions, usage examples, API documentation. Evidence must list which sections were found.

2. **agents_md** (file_existence) — Check for: `AGENTS.md` or `CLAUDE.md` at root documenting commands, conventions, and build steps for agents. FOUND if agent instructions file exists.

3. **agents_md_validation** (ci_workflow) — Check CI workflows for jobs that validate commands documented in AGENTS.md still work. FOUND if CI validates agent instructions.

4. **documentation_freshness** (git_history) — Run `git log -1 --format="%ci" -- README.md AGENTS.md CLAUDE.md CONTRIBUTING.md 2>/dev/null` and check if any key doc was modified within 180 days. FOUND if docs are fresh.

5. **automated_doc_generation** (ci_workflow + config_parsing) — Check for: Sphinx conf.py, MkDocs config, Typedoc config, godoc generation in Makefile, `make docs` target, doc generation CI workflows. FOUND if docs are auto-generated from code.

6. **api_schema_docs** (file_existence) — Check for: `openapi.yaml`/`openapi.json`/`swagger.*`, `.proto` files, GraphQL schema files (`*.graphql`, `schema.graphql`). FOUND if API schemas are documented.

7. **service_flow_documented** (file_existence) — Check for: PlantUML `.puml` files, Mermaid diagrams in docs, `docs/architecture/` directory, architecture diagram files (`.png`, `.svg` in docs). FOUND if architecture/service flow diagrams exist.

8. **skills** (file_existence) — Check for: `.claude/skills/` or `.factory/skills/` directories with skill definitions. FOUND if reusable agent skill definitions exist.

9. **changelog_maintained** (file_existence + doc_content) — Check for: `CHANGELOG.md` or `CHANGES.md` at root with structured entries (dates, version headers, categorized changes following Keep a Changelog or similar format). FOUND if a maintained changelog exists. This is distinct from `release_notes_automation` which checks for automated generation.

10. **doc_examples_tested** (config_parsing + code_search) — Check for: Python `doctest` in test config or `--doctest-modules` flag, Go testable examples (`func Example` in `*_test.go`), JSDoc `@example` tags validated by a test runner, Rust doc-tests (`///` examples compiled by `cargo test`). SKIP if project has no public API or library surface. FOUND if documentation code examples are verified by the test suite.

### Pillar 5: Development Environment (5 criteria)

1. **devcontainer** (file_existence) — Check for: `.devcontainer/devcontainer.json`, `.devcontainer.json`, `flake.nix` with devShell output, `shell.nix`, or `default.nix` with development dependencies. FOUND if devcontainer or Nix-based reproducible development environment config exists.

2. **devcontainer_runnable** (config_parsing) — If devcontainer or Nix dev environment exists, check if it has a valid base image and features configured (devcontainer) or valid inputs and devShell output (Nix flake). SKIP if no devcontainer or Nix config. FOUND if dev environment appears buildable.

3. **env_template** (file_existence) — Check for: `.env.example`, `.env.template`, `.envrc.example`, documented env vars in docs, `.tool-versions` (asdf), `.mise.toml` or `.mise/config.toml` (mise), `.nvmrc`, `.node-version`, `.python-version`, `.ruby-version`, `.go-version`. FOUND if env template or tool version pinning exists.

4. **local_services_setup** (file_existence) — Check for: `docker-compose.yml` or `compose.yml` with service definitions (Postgres, Redis, etc.). FOUND if local services are containerized.

5. **database_schema** (file_existence) — Check for: migration files (Alembic, Prisma, ActiveRecord, Flyway, Goose, Liquibase), `schema.prisma`, `schema.sql`, SQLAlchemy models, JPA/Hibernate entity classes, Ecto migrations (Elixir), sea-orm entities. SKIP if project has no database. FOUND if schema management exists.

### Pillar 6: Debugging & Observability (11 criteria)

1. **structured_logging** (dependency_check) — Check dependency manifests and imports for: `zap`, `logrus`, `zerolog` (Go), `structlog`, `logging` with formatters (Python), `winston`, `pino` (Node.js), `tracing` crate (Rust), `logback` with JSON encoder, `log4j2` with JSON layout, `slf4j` (Java/Kotlin), `Logger` (Elixir). FOUND if structured logging library is used.

2. **distributed_tracing** (dependency_check) — Check for: OpenTelemetry packages, `opentracing`, Jaeger client, `X-Request-ID` middleware, `dd-trace`. FOUND if distributed tracing is instrumented.

3. **metrics_collection** (dependency_check) — Check for: Prometheus client libraries, Datadog StatsD, OpenTelemetry metrics, custom metrics packages. FOUND if metrics are collected.

4. **alerting_configured** (config_parsing) — Check for: Prometheus Alertmanager configs, PagerDuty integration, OpsGenie config, alert rules files. FOUND if alerting is configured.

5. **deployment_observability** (config_parsing + doc_content) — Check for: Grafana dashboard configs, monitoring documentation, deploy notification configs, dashboard links in docs. FOUND if deployment monitoring exists.

6. **error_tracking_contextualized** (dependency_check) — Check for: Sentry SDK with release tracking, Bugsnag, Rollbar, error tracking with context in dependencies. FOUND if error tracking service is configured.

7. **health_checks** (code_search) — Search for: `/health` or `/healthz` endpoint definitions, liveness/readiness probe configs, health check functions. FOUND if health check endpoints exist.

8. **profiling_instrumentation** (code_search + dependency_check) — Check for: `pprof` imports (Go), `py-spy`/`Pyroscope` (Python), profiling middleware, `Performance.md`, tracy (Rust/C++). FOUND if profiling infrastructure exists.

9. **code_quality_metrics** (ci_workflow) — Check for: Codecov integration, SonarQube/SonarCloud config, CodeClimate, coverage upload in CI, Coveralls. FOUND if code quality or coverage tracking is configured via CI. Note: CodeQL is a SAST security tool and is assessed under `automated_security_review` in Pillar 7 — not counted here to avoid double-counting.

10. **circuit_breakers** (dependency_check) — Check for: circuit breaker libraries (`sony/gobreaker`, `resilience4j`, `hystrix`, `cockatiel`), retry-with-backoff patterns. FOUND if resilience patterns are implemented.

11. **runbooks_documented** (file_existence) — Check for: `runbooks/` directory, `INCIDENT_RESPONSE.md`, `PLAYBOOK.md`, operational docs with incident procedures. FOUND if runbooks exist.

### Pillar 7: Security & Governance (13 criteria)

1. **branch_protection** (api_check) — Run `gh api repos/{owner}/{repo}/rulesets 2>/dev/null` or `gh api repos/{owner}/{repo}/branches/main/protection 2>/dev/null`. SKIP if gh unavailable. FOUND if branch protection rules exist.

2. **codeowners** (file_existence) — Check for: `.github/CODEOWNERS`, `CODEOWNERS`, `docs/CODEOWNERS`. FOUND if CODEOWNERS file exists with team assignments.

3. **secret_scanning** (api_check) — Run `gh api repos/{owner}/{repo} --jq '.security_and_analysis.secret_scanning.status' 2>/dev/null`. SKIP if gh unavailable or insufficient permissions. FOUND if secret scanning is enabled.

4. **secrets_management** (config_parsing + code_search) — Check for: references to vault (HashiCorp Vault, AWS Secrets Manager), `secrets.*` patterns in config, GitHub Actions secrets usage, SOPS config, `.envrc` gitignored. FOUND if secrets use a vault or manager.

5. **dependency_update_automation** (file_existence) — Check for: `.github/dependabot.yml`, `renovate.json`, `renovate.json5`, `.renovaterc`. FOUND if dependency update bot is configured.

6. **gitignore_comprehensive** (config_parsing) — Read `.gitignore` and check if it covers: `.env*`, credentials, `node_modules/` or equivalent, build artifacts, IDE configs (`.idea/`, `.vscode/`). FOUND if gitignore properly excludes sensitive files and artifacts.

7. **automated_security_review** (ci_workflow) — Check for: CodeQL analysis workflow, Snyk, Trivy, Semgrep, SAST pipeline steps in CI. FOUND if security scanning runs in CI.

8. **log_scrubbing** (code_search) — Search for: redaction functions, `SafeValue` interfaces, password masking patterns, sanitization middleware, log field filtering. FOUND if sensitive data is redacted from logs.

9. **pii_handling** (code_search) — Search for: data classification annotations, GDPR controls, redactability systems, PII detection tooling. SKIP if project doesn't handle user data. FOUND if PII is classified and handled.

10. **dast_scanning** (ci_workflow) — Check for: OWASP ZAP, Burp Suite, Nuclei in CI workflows, dynamic scanning steps. SKIP if project is a library/CLI. FOUND if DAST runs in CI.

11. **privacy_compliance** (config_parsing) — Check for: data retention policies, consent management, privacy tooling, GDPR/CCPA controls. SKIP if project doesn't collect user data. FOUND if privacy compliance is enforced.

12. **container_image_scanning** (ci_workflow) — Check for: Trivy, Snyk Container, Grype, or Anchore in CI workflows scanning Docker/OCI images for vulnerabilities. Also check for `docker scout` or `cosign` usage. SKIP if project does not build container images (no `Dockerfile`, `Containerfile`, or container build step in CI). FOUND if container image vulnerability scanning runs in CI.

13. **sbom_generation** (ci_workflow + config_parsing) — Check for: `syft`, `cyclonedx-bom`, `spdx-sbom-generator`, `trivy sbom`, or `docker sbom` in CI workflows or build scripts. Also check for SBOM output files (`.spdx.json`, `.cdx.json`, `bom.json`). SKIP if project does not produce distributable artifacts. FOUND if Software Bill of Materials is generated as part of the build or release pipeline.

### Pillar 8: Task Discovery (4 criteria)

1. **issue_templates** (file_existence) — Check for: `.github/ISSUE_TEMPLATE/` directory with template files, `.github/ISSUE_TEMPLATE.md`. FOUND if structured issue templates exist.

2. **issue_labeling_system** (api_check) — Run `gh label list --limit 50 2>/dev/null` and check for organized label taxonomy (priority, type, area labels). SKIP if gh unavailable. FOUND if comprehensive labels exist.

3. **backlog_health** (api_check) — Run `gh issue list --limit 20 --json title,labels 2>/dev/null`. Check if >70% of issues have labels and titles >10 chars. SKIP if gh unavailable. FOUND if backlog is well-maintained. Evidence must include counts: "Found X/Y issues with labels (Z%, threshold: 70%)".

4. **pr_templates** (file_existence) — Check for: `.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE/`, `PULL_REQUEST_TEMPLATE.md`. FOUND if PR template exists with structured sections.

### Pillar 9: Product & Analytics (3 criteria)

1. **product_analytics_instrumentation** (dependency_check) — Check for: Mixpanel, Amplitude, PostHog, Heap, GA4, custom analytics SDKs in dependencies. SKIP if server infrastructure/library. FOUND if analytics is instrumented.

2. **error_to_insight_pipeline** (config_parsing) — Check for: Sentry-GitHub integration, automated issue creation from errors, error tracking to ticket pipeline. FOUND if errors automatically create trackable issues.

3. **experiment_infrastructure** (config_parsing) — Check for: A/B testing framework, feature flags with metrics integration, experiment platform config. FOUND if experiment infrastructure exists.

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

Each criterion has an impact tier that determines its weight in the weighted score:

- **High (weight 3):** type_check, strict_typing, unit_tests_exist, deps_pinned, branch_protection, secrets_management, gitignore_comprehensive, automated_security_review, secret_scanning
- **Medium (weight 2):** All criteria not listed as high or low (including custom criteria with `impact: medium` or unspecified impact)
- **Low (weight 1):** duplicate_code_detection, tech_debt_tracking, naming_consistency, service_flow_documented, test_performance_tracking, profiling_instrumentation, runbooks_documented

Custom criteria from `.agentfit.yml` use their specified `impact` tier. Calculate: `weighted_score = round(sum(passed_criterion_weight) / sum(applicable_criterion_weight) * 100)`. Display alongside the unweighted pass rate.

### Maturity Level Calculation

Each criterion is assigned to a maturity level (L1-L5). Calculate completion percentage for each level using only the criteria assigned to that level.

**Level-to-criteria mapping:**

**Level 1 (Functional):** formatter, lint_config, type_check, strict_typing, unit_tests_exist, unit_tests_runnable, test_naming_conventions, readme, documentation_freshness, build_cmd_doc, deps_pinned, vcs_cli_tools, gitignore_comprehensive

**Level 2 (Documented):** pre_commit_hooks, naming_consistency, agents_md, skills, devcontainer, env_template, local_services_setup, database_schema, single_command_setup, branch_protection, codeowners, secrets_management, issue_templates, pr_templates, structured_logging, changelog_maintained

**Level 3 (Standardized):** large_file_detection, code_modularization, cyclomatic_complexity, dead_code_detection, duplicate_code_detection, tech_debt_tracking, integration_tests_exist, test_coverage_thresholds, test_isolation, agents_md_validation, automated_doc_generation, api_schema_docs, service_flow_documented, doc_examples_tested, distributed_tracing, metrics_collection, health_checks, code_quality_metrics, secret_scanning, dependency_update_automation, automated_security_review, log_scrubbing, container_image_scanning, sbom_generation, issue_labeling_system, backlog_health, fast_ci_feedback, release_automation, release_notes_automation, unused_deps_detection, heavy_dependency_detection

**Level 4 (Optimized):** n_plus_one_detection, deployment_frequency, automated_pr_review, agentic_development, feature_flag_infrastructure, build_performance_tracking, monorepo_tooling, flaky_test_detection, test_performance_tracking, devcontainer_runnable, alerting_configured, deployment_observability, error_tracking_contextualized, profiling_instrumentation, circuit_breakers, runbooks_documented, pii_handling, dast_scanning, privacy_compliance

**Level 5 (Autonomous):** dead_feature_flag_detection, progressive_rollout, rollback_automation, version_drift_detection, product_analytics_instrumentation, error_to_insight_pipeline, experiment_infrastructure

**Gated progression rule:** To unlock level N, the repository must pass ≥80% of applicable criteria at level N AND all previous levels. Calculate each level's completion percentage (excluding skipped criteria). The maturity level is the highest level where the 80% gate is met for that level and all levels below it.

**Level bar weights:** For the progress bar, calculate each level's visual width proportional to its applicable criteria count: `{lN_weight} = round(applicable_criteria_at_level_N / total_applicable_criteria * 100)`. Ensure the 5 weights sum to 100 (adjust the largest to absorb rounding).

### Strengths and Opportunities

**Strengths (top 3):** Select the 3 pillars with the highest percentage scores. For each, list the pillar name with percentage and 2-3 key passing criteria as evidence.

**Opportunities (top 3):** Select the 3 most impactful MISSING criteria, prioritized by maturity level (L1 gaps first, then L2, then L3, etc.). Within a level, prioritize criteria from the pillar with the lowest pass rate. Within the same pillar, prioritize alphabetically by criterion name. For each, provide the criterion name, a specific remediation action, and a concrete fix instruction (e.g., "Create `.pre-commit-config.yaml` with a formatter hook", "Add `dependabot.yml` with weekly update schedule", "Create `AGENTS.md` documenting build commands and test workflow"). The fix instruction should be immediately actionable — a single file to create or config to add.

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

After writing the file, open it with: `open /tmp/agentfit-report-{project_name}.html 2>/dev/null || xdg-open /tmp/agentfit-report-{project_name}.html 2>/dev/null || echo "Report saved to /tmp/agentfit-report-{project_name}.html"`

Also write the assessment data as JSON to `/tmp/agentfit-report-{project_name}.json` with this structure:
```json
{
  "schema_version": "1.0.0",
  "skill_version": "1.0.0",
  "project_name": "{project_name}",
  "timestamp": "{ISO 8601 timestamp}",
  "pass_rate": {pass_rate},
  "weighted_score": {weighted_score},
  "maturity_level": {maturity_level},
  "criteria": {
    "{criterion_name}": { "status": "found|missing|skipped", "evidence": "..." }
  }
}
```
This JSON file serves as the grounding reference for future evaluations.

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
    color: #999;
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

  .l1 { background: #1a6b3a; }
  .l2 { background: #2d8a4e; }
  .l3 { background: #40a862; }
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
    font-weight: 400;
    letter-spacing: 2px;
    text-transform: uppercase;
    color: #999;
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
  .highlight-num.green { color: #4caf50; }
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
    color: #999;
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
    color: #999;
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
    color: #999;
    line-height: 1.4;
    word-wrap: break-word;
    overflow-wrap: break-word;
  }

  @media (max-width: 768px) {
    body { padding: 20px 16px; }
    h1 { font-size: 2rem; }
    .columns { grid-template-columns: 1fr; gap: 32px; }
    .criterion-row {
      grid-template-columns: 24px 1fr;
      gap: 8px;
    }
    .criterion-score { text-align: left; }
    .criterion-evidence { grid-column: 1 / -1; padding-left: 32px; }
    .level-labels { font-size: 0.65rem; }
  }
</style>
</head>
<body>

<header>
  <h1>{project_name} <span class="badge">{language}</span></h1>
  <p class="meta">{repo_path}&nbsp;&nbsp;&nbsp;PASS RATE {pass_rate}%&nbsp;&nbsp;·&nbsp;&nbsp;WEIGHTED {weighted_score}%</p>
  <p class="description">{project_description}</p>
</header>

<main>
  <!-- LEVEL PROGRESS BAR -->
  <div class="level-bar" role="progressbar" aria-valuenow="{maturity_level}" aria-valuemin="1" aria-valuemax="5" aria-label="Maturity level {maturity_level} of 5">
    <div class="level-segment l1" style="width:{l1_weight}%"></div>
    <div class="level-segment l2" style="width:{l2_weight}%"></div>
    <div class="level-segment l3" style="width:{l3_weight}%"></div>
    <div class="level-segment l4" style="width:{l4_weight}%"></div>
    <div class="level-segment l5" style="width:{l5_weight}%"></div>
  </div>
  <div class="level-labels" aria-hidden="true">
    <div class="level-label" style="width:{l1_weight}%"><span class="pct">{l1_pct}%</span>&nbsp;<span class="name">L1</span></div>
    <div class="level-label" style="width:{l2_weight}%"><span class="pct">{l2_pct}%</span>&nbsp;<span class="name">L2</span></div>
    <div class="level-label" style="width:{l3_weight}%"><span class="pct">{l3_pct}%</span>&nbsp;<span class="name">L3</span></div>
    <div class="level-label" style="width:{l4_weight}%"><span class="pct">{l4_pct}%</span>&nbsp;<span class="name">L4</span></div>
    <div class="level-label" style="width:{l5_weight}%"><span class="pct">{l5_pct}%</span>&nbsp;<span class="name">L5</span></div>
  </div>

  <!-- SUMMARY -->
  <section class="summary-section">
    <h2>{summary_headline}</h2>
    <p>{summary_text}</p>

    <div class="columns">
      <div>
        <h3 class="col-header">STRENGTHS</h3>
        <!-- Repeat for each strength (top 3) -->
        <article class="highlight">
          <div class="highlight-num green">01</div>
          <div class="highlight-title">{strength_1_title}</div>
          <div class="highlight-detail">{strength_1_detail}</div>
        </article>
        <!-- 02, 03 ... -->
      </div>
      <div>
        <h3 class="col-header">OPPORTUNITIES</h3>
        <!-- Repeat for each opportunity (top 3) -->
        <article class="highlight">
          <div class="highlight-num orange">01</div>
          <div class="highlight-title">{opportunity_1_title}</div>
          <div class="highlight-detail">{opportunity_1_detail}</div>
        </article>
        <!-- 02, 03 ... -->
      </div>
    </div>
  </section>

  <!-- ALL CRITERIA -->
  <section class="criteria-section">
    <h2 class="criteria-header">ALL CRITERIA</h2>

    <!-- Repeat for each pillar (9 total) -->
    <section class="pillar-group">
      <div class="pillar-header">
        <span class="pillar-name">{pillar_name}</span>
        <span class="pillar-score">{passed}/{total} ({percentage}%)</span>
      </div>

      <!-- Repeat for each criterion in this pillar, alphabetically by name -->
      <div class="criterion-row">
        <span class="status-icon {pass|fail|skip}" aria-label="{pass: Passed|fail: Failed|skip: Skipped}">{✓|✗|—}</span>
        <span class="criterion-name">{criterion_name}</span>
        <span class="criterion-score">{1/1|0/1|—/—}</span>
        <span class="criterion-evidence">{evidence_text}</span>
      </div>
    </section>
  </section>
</main>

<footer style="margin-top:48px;padding-top:24px;border-top:1px solid #1a1a2e;font-size:0.75rem;color:#666;font-family:monospace;">
  Agent Fit v1.0.0 · {assessment_date} · {git_sha} · {total_evaluated} criteria evaluated · {total_skipped} skipped
</footer>

</body>
</html>
```

**IMPORTANT**: When generating the HTML, replace ALL `{placeholder}` values with actual data from the assessment. Do NOT leave any placeholders in the output. Generate the complete HTML string and write it to the temp file in a single Bash command.

## Step 5: Report to User

After opening the HTML report, print a brief summary to the console:

```
Agent Fit Report: {project_name}
Pass Rate: {pass_rate}% | Weighted: {weighted_score}% | Level: L{maturity_level} ({maturity_label})
Report: /tmp/agentfit-report-{project_name}.html

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

