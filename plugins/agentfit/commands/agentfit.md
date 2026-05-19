---
description: Evaluates codebase agent-readiness across 9 pillars, 80+ criteria, and 5 maturity levels — produces an HTML report with pass rate scoring
---

When invoked, perform the following assessment. This is a READ-ONLY analysis — do NOT modify any files in the target codebase. The only files you create are the HTML report and its JSON sidecar.

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
6. Detect if the repository is a fork:
   - Run `gh api repos/{owner}/{repo} --jq '.fork' 2>/dev/null`
   - If `true`, extract upstream via `gh api repos/{owner}/{repo} --jq '.parent.full_name' 2>/dev/null`
   - Record `is_fork: true/false` and `upstream_repo` if applicable
   - When fork is detected: for GitHub API criteria (`branch_protection`, `secret_scanning`, `backlog_health`, `issue_labeling_system`), evaluate against the upstream repo instead, or skip with evidence "Fork detected; criterion evaluated against upstream {upstream_repo}"

### Grounding on Previous Reports

If a previous report exists at `/tmp/agentfit-report-{project_name}.json`, load it and use its criterion statuses as a reference baseline. When evaluating each criterion:
- If the previous status was `found` and the current evidence still supports it, maintain `found`
- If the previous status was `missing` and no new evidence is found, maintain `missing`
- Only change a criterion's status if the underlying signal has demonstrably changed (e.g., a config file was added or removed)
- In the evidence field, note when a status changed from the previous report: "Changed from missing → found: [reason]"

If no previous report exists, evaluate all criteria fresh.
If the previous report exists but is unreadable or malformed JSON, continue with fresh evaluation, note baseline unavailability in the report metadata/delta section, and do not fail the assessment run.

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
- **Kubernetes operator**: Check for `controller-runtime` or `operator-sdk` in go.mod, CRD manifests in `config/crd/` or `install/`, OWNERS files at root, `.ci-operator.yaml`, or `kubebuilder` markers in source. If found, mark as k8s-operator.
- **Mobile app**: Check for `ios/` or `android/` directories, `Podfile`, `build.gradle` with Android SDK, `*.xcodeproj`, or Flutter `pubspec.yaml`. If found, mark as mobile.
- **Infrastructure/IaC**: Check for `*.tf` files (Terraform), `Pulumi.yaml`, CloudFormation templates, or Ansible playbooks as the primary project type. If found, mark as iac.

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
- It checks for `devcontainer`/`env_template`/`local_services_setup` but the project is a Kubernetes operator (operators need a cluster, not docker-compose; substitute: check for envtest config, kind/minikube setup scripts, or e2e cluster provisioning in Makefile/hack/)
- It checks for `circuit_breakers` but the project is a Kubernetes operator (client-go provides built-in retry/backoff via informers and work queues)
- It checks for `error_tracking_contextualized` (Sentry) but the project is a Kubernetes operator or CLI tool (operators report errors via k8s Conditions and Events; CLI tools report errors to stderr)
- It checks for `secrets_management` (Vault/SOPS) but the project is a Kubernetes operator (operators manage secrets via k8s Secrets API, not external vaults)
- It checks for `health_checks`/`alerting_configured`/`metrics_collection`/`deployment_observability`/`distributed_tracing` but the project is a CLI tool (CLI tools are invoked on-demand, not long-running services)
- It checks for `progressive_rollout`/`rollback_automation` but the project is a CLI tool or IaC project (these are distributed via package managers, not deployed as services)

When skipping, always explain why in the evidence field.

### Pillar 1: Style & Validation (13 criteria)

Evaluate each criterion. For all config-parsing criteria, read the actual config file and report what rules/settings are configured.

1. **formatter** (config_parsing) — Check for: `.prettierrc*`, `biome.json`, `pyproject.toml` with `[tool.black]` or `[tool.ruff.format]`, `rustfmt.toml`, `.clang-format`, `.editorconfig` with formatting rules, `gofumpt`/`goimports` in Makefile or CI, `google-java-format` or `spotless` in Gradle/Maven config, `ktlint` config, `mix format` config (Elixir). FOUND if any formatter is configured.

2. **lint_config** (config_parsing) — Check for: `.eslintrc*`, `eslint.config.*`, `biome.json` with linter, `.golangci.yml`, `pyproject.toml` with `[tool.ruff]` or `[tool.pylint]`, `clippy.toml` or `Cargo.toml` with clippy config, `.clang-tidy`, Checkstyle config (`checkstyle.xml`), PMD ruleset (`pmd.xml`), SpotBugs/FindBugs config, `detekt.yml` (Kotlin), `credo` config (Elixir). FOUND if linter configured with specific rules.

3. **type_check** (config_parsing) — Check for: `tsconfig.json`, `mypy.ini` or `pyproject.toml` with `[tool.mypy]`, Go compiler (always passes for Go), Rust compiler (always passes for Rust), Java compiler (always passes for Java/Kotlin), `pyrightconfig.json`. FOUND if type checking is configured.

4. **strict_typing** (config_parsing) — Check for: `tsconfig.json` with `"strict": true`, `mypy` with `strict = true` or `disallow_untyped_defs = true`, Go (always strict), Rust (always strict), Java (always strict), Kotlin (always strict), Clippy pedantic. FOUND if strict mode is enabled.

5. **pre_commit_hooks** (file_existence) — Check for: `.pre-commit-config.yaml`, `.husky/` directory, `.lefthook.yml`, `lint-staged` in `package.json`, `scripts/pre-commit*`. FOUND if any pre-commit hook framework is configured.

6. **naming_consistency** (config_parsing + doc_content) — Check for: ESLint `naming-convention` rule, `revive` linter naming rules in golangci-lint, documented naming conventions in AGENTS.md/CLAUDE.md/CONTRIBUTING.md, `.editorconfig`. FOUND if naming conventions are documented or enforced via linter.

7. **large_file_detection** (file_existence) — Check for: `.gitattributes` with Git LFS `filter=lfs` entries (not `linguist-generated` or `linguist-vendored` markers, which are for GitHub language stats), `pre-commit` hook `check-added-large-files`, CI jobs checking file size. FOUND if large file detection is configured.

8. **code_modularization** (config_parsing) — Check for: Go `internal/` packages, `eslint-plugin-boundaries`, Nx boundaries config, Rust module visibility, monorepo workspace boundaries. FOUND if module boundary enforcement exists.

9. **cyclomatic_complexity** (config_parsing) — Check for: `gocyclo` or `cyclop` in golangci-lint config, ESLint `complexity` rule, Ruff `C901`, Clippy `cognitive_complexity`. FOUND if complexity analysis tools are configured.

10. **dead_code_detection** (config_parsing) — Check for: `knip` in package.json (knip code analysis mode — detects unused exports, files, and types), `vulture` in Python config, `staticcheck` unused checks, Ruff `F401`/`F841`, Clippy `dead_code` warnings, `deadcode` tool. FOUND if dead code detection is configured.

11. **duplicate_code_detection** (config_parsing) — Check for: `jscpd` config or in CI, `PMD CPD`, `Simian`, golangci-lint `dupl` linter, SonarQube duplication. FOUND if duplication scanner is configured.

12. **tech_debt_tracking** (ci_workflow) — Check for: CI workflows scanning for TODO/FIXME markers, `godox` linter in golangci-lint, SonarQube, `check-todos` scripts. FOUND if tech debt tracking is automated.

13. **n_plus_one_detection** (config_parsing) — Check for: `bullet` gem (Ruby), `django-auto-prefetch` or `nplusone` (Python), Prisma query warnings, ORM query analyzers. SKIP if project doesn't use an ORM or database. FOUND if N+1 detection is configured.

### Pillar 2: Build System (19 criteria)

1. **build_cmd_doc** (doc_content) — Check README.md, AGENTS.md, CLAUDE.md, Makefile, Taskfile.yml, Justfile, Earthfile, CONTRIBUTING.md for a code block or command documenting build/install/run instructions. FOUND if at least one of these files contains a code block or target with build/run/install instructions. Evidence must cite which file and what command was found.

2. **deps_pinned** (file_existence) — Check for: `go.sum`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `uv.lock`, `poetry.lock`, `Cargo.lock`, `Pipfile.lock`, `gradle.lockfile` or `buildSrc/*.lock`, `mix.lock` (Elixir). FOUND if lockfile exists. Note: Rust libraries intentionally gitignore Cargo.lock — skip for Rust library projects. Note: Maven/Gradle projects often pin versions in POM/build files directly — check for version pinning in `pom.xml` or `build.gradle` if no lockfile.

3. **vcs_cli_tools** (doc_content + file_existence) — Check for: VCS workflow tooling documented in Makefile, CONTRIBUTING.md, or README.md (e.g., PR creation commands, branch policies), OR `.github/workflows/*.yml` referencing `gh` CLI commands, OR Prow OWNERS files with documented `/lgtm`, `/approve` workflow, OR `.gitlab-ci.yml` with merge request automation. FOUND if VCS tooling usage is documented or automated in CI.

4. **single_command_setup** (doc_content) — Check README.md, AGENTS.md, CONTRIBUTING.md for a code block with a single setup command (e.g., `make setup`, `task setup`, `just setup`, `earthly +setup`, `docker-compose up`, `npm install`, `nix develop`, `./dev doctor`) that completes project setup (not a multi-step procedure). FOUND if a single setup command is documented in a code block. Evidence must quote the command found.

5. **fast_ci_feedback** (ci_workflow) — Read CI config files: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`, `.buildkite/pipeline.yml`, `Jenkinsfile`, `.ci-operator.yaml`. FOUND if CI uses caching (actions/cache, turbo, sccache, `Cache@2` in Azure Pipelines, or language-specific caching) AND does not include matrix > 5 targets without parallelization. Evidence must cite specific caching actions and matrix size found.

6. **deployment_frequency** (git_history) — Run `git tag --sort=-creatordate | head -20` and check dates. FOUND if multiple releases per month or regular release cadence visible.

7. **release_automation** (ci_workflow) — Check for: `.github/workflows/release*`, `goreleaser.yml`, `.releaserc` or `release.config.*` (semantic-release), `release-please-config.json` (release-please), `cargo-release` config, publish workflows, tag-triggered release workflows, `.ci-operator.yaml` with release pipeline config, `.gitlab-ci.yml` with deploy stages. FOUND if releases are automated via CI.

8. **release_notes_automation** (config_parsing) — Check for: `.goreleaser.yml` with changelog, `towncrier.toml`, `.changeset/`, `git-cliff.toml`, `cliff.toml`, release-changelog-builder action in CI, `release-please-config.json`, `.releaserc` with changelog plugin. FOUND if release notes are auto-generated.

9. **automated_pr_review** (ci_workflow + code_search) — Check for: CodeRabbit, Copilot Code Review, Danger.js, custom review bots in CI, Semgrep in PR checks, CodeAnt AI, Graphite Reviewer, Sourcery, `claude-code` review in CI. FOUND if automated review comments are generated on PRs.

10. **agentic_development** (git_history) — Run `git log --format="%aN" -100 2>/dev/null` and check for `Co-authored-by:` trailers or author names matching: Claude, Copilot, factory-droid, Devin, Sweep, cursor-bot. Merge bots (`dependabot`, `renovate-bot`, `openshift-merge-bot`) do NOT count as agentic development — they are automation, not AI coding agents. FOUND if agent co-authorship is visible in recent git history. Note: file-existence checks for AGENTS.md and .claude/skills/ are already covered by the `agents_md` and `skills` criteria in Pillar 4 — this criterion only checks the unique signal of agents actively contributing code.

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

3. **agents_md_validation** (ci_workflow) — Check any CI config (GitHub Actions, GitLab CI, Azure Pipelines, Prow, etc.) for jobs that validate agent instructions documented in AGENTS.md. FOUND if CI validates agent instructions.

4. **documentation_freshness** (git_history) — Run `git log -1 --format="%ci" -- README.md AGENTS.md CLAUDE.md CONTRIBUTING.md docs/ 2>/dev/null` and check if any key doc or `docs/` directory was modified within 180 days. FOUND if docs are fresh.

5. **automated_doc_generation** (ci_workflow + config_parsing) — Check for: Sphinx conf.py, MkDocs config, Typedoc config, godoc generation in Makefile, `make docs` target, doc generation CI workflows. FOUND if docs are auto-generated from code.

6. **api_schema_docs** (file_existence) — Check for: `openapi.yaml`/`openapi.json`/`swagger.*`, `.proto` files, GraphQL schema files (`*.graphql`, `schema.graphql`). FOUND if API schemas are documented.

7. **service_flow_documented** (file_existence) — Check for: PlantUML `.puml` files, Mermaid diagrams in docs, `docs/architecture/` directory, architecture diagram files (`.png`, `.svg` in docs). FOUND if architecture/service flow diagrams exist.

8. **skills** (file_existence) — Check for: `.claude/skills/` or `.factory/skills/` directories with skill definitions. FOUND if reusable agent skill definitions exist.

9. **changelog_maintained** (file_existence + doc_content) — Check for: `CHANGELOG.md` or `CHANGES.md` at root with structured entries (dates, version headers, categorized changes following Keep a Changelog or similar format). FOUND if a maintained changelog exists. This is distinct from `release_notes_automation` which checks for automated generation.

10. **doc_examples_tested** (config_parsing + code_search) — Check for: Python `doctest` in test config or `--doctest-modules` flag, Go testable examples (`func Example` in `*_test.go`), JSDoc `@example` tags validated by a test runner, Rust doc-tests (`///` examples compiled by `cargo test`). SKIP if project has no public API or library surface. FOUND if documentation code examples are verified by the test suite.

### Pillar 5: Development Environment (5 criteria)

1. **devcontainer** (file_existence) — Check for: `.devcontainer/devcontainer.json`, `.devcontainer.json`, `flake.nix` with devShell output, `shell.nix`, or `default.nix` with development dependencies. FOUND if devcontainer or Nix-based reproducible development environment config exists.

2. **devcontainer_runnable** (config_parsing) — If devcontainer or Nix dev environment exists, check if it has a valid base image and features configured (devcontainer) or valid inputs and devShell output (Nix flake). SKIP if no devcontainer or Nix config. FOUND if dev environment appears buildable.

3. **env_template** (file_existence) — Check for: `.env.example`, `.env.template`, `.envrc.example`, documented env vars in docs, `.tool-versions` (asdf), `.mise.toml` or `.mise/config.toml` (mise), `.nvmrc`, `.node-version`, `.python-version`, `.ruby-version`, `.go-version`. For Go projects, also accept Go version pinned in `go.mod`. For Kubernetes operators, accept envtest setup in Makefile (e.g., `setup-envtest` target). FOUND if env template or tool version pinning exists.

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

2. **codeowners** (file_existence) — Check for: `.github/CODEOWNERS`, `CODEOWNERS`, `docs/CODEOWNERS`, OR Prow OWNERS files at root with `approvers`/`reviewers` fields (Kubernetes ecosystem equivalent). FOUND if any code ownership file exists with team assignments.

3. **secret_scanning** (api_check) — Run `gh api repos/{owner}/{repo} --jq '.security_and_analysis.secret_scanning.status' 2>/dev/null`. SKIP if gh unavailable or insufficient permissions. FOUND if secret scanning is enabled.

4. **secrets_management** (config_parsing + code_search) — Check for: references to vault (HashiCorp Vault, AWS Secrets Manager), `secrets.*` patterns in config, GitHub Actions secrets usage, SOPS config, `.envrc` gitignored. FOUND if secrets use a vault or manager.

5. **dependency_update_automation** (file_existence) — Check for: `.github/dependabot.yml`, `renovate.json`, `renovate.json5`, `.renovaterc`. FOUND if dependency update bot is configured.

6. **gitignore_comprehensive** (config_parsing) — Read `.gitignore` and check if it covers language-appropriate patterns: Go: `_output/`, IDE configs (`.idea/`, `.vscode/`), `*.test` binaries; JS/TS: `.env*`, `node_modules/`, `dist/`, IDE configs; Python: `.env*`, `__pycache__/`, `*.egg-info/`, `.venv/`, IDE configs; Rust: `target/`, IDE configs. FOUND if gitignore covers build artifacts + IDE configs for the detected language. `.env*` only required for languages that use dotenv (JS/TS, Python, Ruby).

7. **automated_security_review** (ci_workflow) — Check for: CodeQL analysis workflow, Snyk, Trivy, Semgrep, SAST pipeline steps in CI, `gosec` linter enabled in `.golangci.yml` (Go SAST), security scanning steps in any CI config (not just GitHub Actions). FOUND if security scanning runs in CI.

8. **log_scrubbing** (code_search) — Search for: redaction functions, `SafeValue` interfaces, password masking patterns, sanitization middleware, log field filtering. FOUND if sensitive data is redacted from logs.

9. **pii_handling** (code_search) — Search for: data classification annotations, GDPR controls, redactability systems, PII detection tooling. SKIP if project doesn't handle user data. FOUND if PII is classified and handled.

10. **dast_scanning** (ci_workflow) — Check for: OWASP ZAP, Burp Suite, Nuclei in CI workflows, dynamic scanning steps. SKIP if project is a library/CLI. FOUND if DAST runs in CI.

11. **privacy_compliance** (config_parsing) — Check for: data retention policies, consent management, privacy tooling, GDPR/CCPA controls. SKIP if project doesn't collect user data. FOUND if privacy compliance is enforced.

12. **container_image_scanning** (ci_workflow) — Check for: Trivy, Snyk Container, Grype, or Anchore in CI workflows scanning Docker/OCI images for vulnerabilities. Also check for `docker scout` or `cosign` usage. SKIP if project does not build container images (no `Dockerfile`, `Containerfile`, or container build step in CI). FOUND if container image vulnerability scanning runs in CI.

13. **sbom_generation** (ci_workflow + config_parsing) — Check for: `syft`, `cyclonedx-bom`, `spdx-sbom-generator`, `trivy sbom`, or `docker sbom` in CI workflows or build scripts. Also check for SBOM output files (`.spdx.json`, `.cdx.json`, `bom.json`). SKIP if project does not produce distributable artifacts. FOUND if Software Bill of Materials is generated as part of the build or release pipeline.

### Pillar 8: Task Discovery (4 criteria)

1. **issue_templates** (file_existence) — Check for: `.github/ISSUE_TEMPLATE/` directory with template files, `.github/ISSUE_TEMPLATE.md`. FOUND if structured issue templates exist.

2. **issue_labeling_system** (api_check) — Run `gh label list --limit 50 2>/dev/null` and check for organized label taxonomy (priority, type, area labels). GitHub's 9 default labels (`bug`, `documentation`, `duplicate`, `enhancement`, `good first issue`, `help wanted`, `invalid`, `question`, `wontfix`) alone do NOT satisfy this criterion. FOUND only if custom labels beyond defaults exist (e.g., `priority/*`, `area/*`, `kind/*` labels). SKIP if gh unavailable.

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

**Level gate cards (HTML + console):** For each level N, set:
- `{lN_gate_class}`: `pass` if level N meets the 80% gate and all prior levels pass; `fail` if evaluated but below gate; `blocked` if a prior level gate is not met
- `{lN_gate_status}`: `✓ Passed` | `✗ Below gate` | `Blocked by L{m}` (use the lowest unmet prerequisite level for m)
- `{lN_gate_symbol}` (console only): `✓` | `✗` | `— (blocked by L{m})`

### Strengths and Opportunities

**Strengths (top 3):** Select the 3 pillars with the highest percentage scores. For each, list the pillar name with percentage and 2-3 key passing criteria as evidence.

**Opportunities (top 3):** Select the 3 most impactful MISSING criteria, prioritized by maturity level (L1 gaps first, then L2, then L3, etc.). Within a level, prioritize criteria from the pillar with the lowest pass rate. Within the same pillar, prioritize alphabetically by criterion name. For each, provide the criterion name, a specific remediation action, and a concrete fix instruction (e.g., "Create `.pre-commit-config.yaml` with a formatter hook", "Add `dependabot.yml` with weekly update schedule", "Create `AGENTS.md` documenting build commands and test workflow"). The fix instruction should be immediately actionable — a single file to create or config to add.

### Remediation Roadmap

Generate a complete remediation roadmap from ALL missing criteria (not just the top 3 opportunities), grouped by maturity level (L1 → L5). Skip levels with no missing criteria. For each missing criterion, provide:
- **criterion**: snake_case name
- **pillar**: parent pillar name
- **impact**: high / medium / low (from the weighted scoring tiers)
- **fix**: A single, concrete, immediately actionable instruction tailored to the project's detected primary language

Language-aware fix instructions — use the detected language to suggest the correct tool and config format:
- **Go**: `.golangci.yml` configs, `go test ./...`, `go.sum`, Makefile targets
- **Python**: `pyproject.toml` with `[tool.ruff]`/`[tool.mypy]`/`[tool.pytest]`, `requirements.txt`
- **TypeScript/JavaScript**: `eslint.config.mjs`, `tsconfig.json` with `strict: true`, `jest.config.ts`, `package-lock.json`
- **Rust**: `clippy.toml`, `cargo test`, `Cargo.lock`, `rustfmt.toml`
- **Java/Kotlin**: `checkstyle.xml`, `build.gradle` configs, JUnit, `pom.xml` settings
- **C++**: `.clang-format`, `.clang-tidy`, `CMakeLists.txt` test targets, `CMakePresets.json`

Architecture-aware fix instructions — use the detected project type to avoid suggesting irrelevant tooling:
- **CLI tools**: Do not suggest health check endpoints, metrics collection, distributed tracing, alerting, or deployment observability. Suggest CLI-appropriate alternatives (e.g., `--version` flag, shell completion scripts, man pages).
- **Kubernetes operators**: Do not suggest docker-compose, `.env` templates, Sentry, or external vault. Suggest k8s-native equivalents (e.g., envtest setup, k8s Conditions/Events for error reporting, k8s Secrets API).
- **Libraries**: Do not suggest deployment, rollout, or service monitoring criteria. Focus on API docs, test coverage, and packaging.
- **IaC projects**: Do not suggest progressive rollout or rollback automation for the project itself. Focus on validation, linting (`tflint`, `ansible-lint`), and state management.

Each fix must be a single file to create or config block to add — not a multi-step procedure. Remediation is strictly advisory — do NOT modify the target codebase.

Include the roadmap in both the HTML report (as a section after ALL CRITERIA) and the JSON output (as a `remediation` array).

### Summary Headline

Generate a short headline based on the strongest pillar or most notable pattern. Examples:
- "Strong Testing" (if Testing pillar is highest)
- "Well-Documented" (if Documentation pillar is highest)
- "Security-First" (if Security pillar is highest)
- "Needs Foundation" (if Level 1 criteria are failing)
- "Automation Momentum" (if CI/release pillars are improving)
- "Observability Upgrade" (if monitoring criteria are strongest)
- "Quality Controls Solid" (if style/testing balance is strong)
- "Governance Gaps Remain" (if security/governance lags)
- "Platform Maturing" (if L2/L3 nearing gate)
- "Readiness Improving" (if delta is positive but gates still blocked)

Then write a 1-2 sentence summary: "{project_name} reaches Level {N} ({maturity_label}) with {total_passed}/{total_applicable} criteria passing. Key areas for improvement include the opportunities listed below."

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
  },
  "remediation": [
    { "criterion": "{name}", "level": {N}, "pillar": "{pillar_name}", "impact": "{high|medium|low}", "fix": "{language_aware_instruction}" }
  ]
}
```
This JSON file serves as the grounding reference for future evaluations and MUST be written on every successful run alongside the HTML report.

### Report Metadata (footer + JSON)

Before generating HTML, capture:
- **assessment_date**: ISO 8601 UTC timestamp when the run completes
- **plugin_version**: Read `version` from `plugins/agentfit/.claude-plugin/plugin.json` (currently `1.0.0`)
- **git_sha**: `git -C {repo} rev-parse --short HEAD 2>/dev/null` or `unknown` if not a git repo
- **assessment_duration**: Elapsed seconds since Step 1 started, formatted as `{N}s` or `{m}m {s}s`
- **total_applicable** / **criteria_skipped**: Counts from the evaluation pass

### HTML Template

The HTML report MUST use this structure with inline CSS and inline JavaScript (zero external dependencies). Replace all `{placeholder}` values with actual assessment data. Each criterion row MUST include `data-status`, `data-level`, and `data-name` attributes for filtering.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{project_name} — Agent Fit Report</title>
<style>
  :root {
    --bg: #07070f;
    --surface: rgba(18, 22, 40, 0.72);
    --border: rgba(120, 130, 180, 0.18);
    --text: #e8eaf6;
    --muted: #8b93b3;
    --accent: #6ee7a8;
    --warn: #f0b429;
    --fail: #ff6b4a;
    --skip: #5a6078;
  }
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
    background: var(--bg);
    color: var(--text);
    line-height: 1.55;
    min-height: 100vh;
    padding: 32px clamp(16px, 4vw, 64px) 48px;
  }
  body::before {
    content: '';
    position: fixed;
    inset: 0;
    background:
      radial-gradient(ellipse 80% 50% at 10% -10%, rgba(110, 231, 168, 0.12), transparent),
      radial-gradient(ellipse 60% 40% at 90% 0%, rgba(99, 102, 241, 0.14), transparent),
      radial-gradient(ellipse 50% 30% at 50% 100%, rgba(240, 180, 41, 0.08), transparent);
    pointer-events: none;
    z-index: 0;
  }
  .page { position: relative; z-index: 1; max-width: 1180px; margin: 0 auto; }

  .hero {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 28px;
    align-items: center;
    padding: 28px 32px;
    margin-bottom: 28px;
    border-radius: 20px;
    border: 1px solid var(--border);
    background: var(--surface);
    backdrop-filter: blur(12px);
    box-shadow: 0 24px 80px rgba(0, 0, 0, 0.45);
  }
  h1 {
    font-size: clamp(1.8rem, 4vw, 2.75rem);
    font-weight: 300;
    letter-spacing: -0.02em;
    color: #fff;
  }
  .badge {
    display: inline-block;
    font-size: 0.72rem;
    font-weight: 700;
    padding: 4px 12px;
    margin-left: 10px;
    border-radius: 999px;
    vertical-align: middle;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    background: linear-gradient(135deg, rgba(110,231,168,0.2), rgba(99,102,241,0.25));
    border: 1px solid rgba(110, 231, 168, 0.35);
    color: var(--accent);
  }
  .meta {
    font-family: ui-monospace, 'SF Mono', Consolas, monospace;
    font-size: 0.8rem;
    color: var(--muted);
    margin-top: 8px;
  }
  .description { font-size: 0.9rem; color: var(--muted); margin-top: 10px; max-width: 52ch; }

  .maturity-badge {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-width: 100px;
    padding: 16px 20px;
    border-radius: 14px;
    border: 1px solid var(--border);
    background: rgba(12, 14, 28, 0.65);
  }
  .maturity-badge .level-num { font-size: 2rem; font-weight: 700; color: #fff; line-height: 1; }
  .maturity-badge .level-sub { font-size: 0.65rem; text-transform: uppercase; letter-spacing: 0.12em; color: var(--muted); margin-top: 4px; }

  .stat-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
    gap: 12px;
    margin-bottom: 28px;
  }
  .stat-card {
    padding: 14px 16px;
    border-radius: 14px;
    border: 1px solid var(--border);
    background: var(--surface);
  }
  .stat-card .label { font-size: 0.65rem; text-transform: uppercase; letter-spacing: 0.1em; color: var(--muted); }
  .stat-card .value { font-size: 1.35rem; font-weight: 600; font-family: ui-monospace, monospace; margin-top: 4px; }
  .stat-card.accent .value { color: var(--accent); }
  .stat-card.warn .value { color: var(--warn); }

  .level-section { margin-bottom: 36px; }
  .level-bar {
    display: flex;
    height: 10px;
    border-radius: 999px;
    overflow: hidden;
    margin-bottom: 10px;
    background: rgba(255,255,255,0.06);
    box-shadow: inset 0 1px 3px rgba(0,0,0,0.4);
  }
  .level-segment { height: 100%; }
  .level-labels {
    display: flex;
    margin-bottom: 16px;
    font-size: 0.72rem;
    font-family: ui-monospace, monospace;
  }
  .level-label { display: flex; align-items: center; gap: 4px; min-width: 0; }
  .level-label .pct { color: #cdd3ef; font-weight: 600; }
  .level-label .name { color: var(--muted); }
  .l1 { background: linear-gradient(90deg, #14532d, #22c55e); }
  .l2 { background: linear-gradient(90deg, #166534, #4ade80); }
  .l3 { background: linear-gradient(90deg, #3f6212, #a3e635); }
  .l4 { background: linear-gradient(90deg, #854d0e, #fbbf24); }
  .l5 { background: linear-gradient(90deg, #374151, #6b7280); }

  .level-gates {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    gap: 8px;
  }
  .gate-card {
    padding: 10px 8px;
    border-radius: 12px;
    border: 1px solid var(--border);
    background: rgba(12, 14, 28, 0.6);
    text-align: center;
    font-size: 0.68rem;
  }
  .gate-card .gate-pct { font-family: ui-monospace, monospace; font-size: 1rem; font-weight: 700; display: block; }
  .gate-card .gate-name { color: var(--muted); margin-top: 2px; display: block; }
  .gate-card .gate-status { margin-top: 6px; font-weight: 600; }
  .gate-card.pass { border-color: rgba(110, 231, 168, 0.45); }
  .gate-card.pass .gate-status { color: var(--accent); }
  .gate-card.fail { border-color: rgba(255, 107, 74, 0.4); }
  .gate-card.fail .gate-status { color: var(--fail); }
  .gate-card.blocked { opacity: 0.65; }
  .gate-card.blocked .gate-status { color: var(--muted); }

  .summary-section { margin-bottom: 40px; }
  .summary-section h2 { font-size: 1.65rem; font-weight: 600; color: #fff; margin-bottom: 12px; }
  .summary-section > p {
    font-family: ui-monospace, monospace;
    font-size: 0.82rem;
    color: var(--muted);
    max-width: 720px;
    margin-bottom: 28px;
  }
  .delta-chip {
    font-family: ui-monospace, monospace;
    font-size: 0.75rem;
    color: var(--warn);
    margin-left: 10px;
    padding: 2px 8px;
    border-radius: 6px;
    background: rgba(240, 180, 41, 0.12);
  }
  .columns { display: grid; grid-template-columns: 1fr 1fr; gap: 40px; }
  .col-header {
    font-size: 0.68rem;
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: var(--muted);
    margin-bottom: 16px;
  }
  .highlight {
    padding: 14px 16px;
    margin-bottom: 12px;
    border-radius: 12px;
    border: 1px solid var(--border);
    background: rgba(12, 14, 28, 0.5);
  }
  .highlight-num { font-size: 0.75rem; font-weight: 800; margin-bottom: 6px; }
  .highlight-num.green { color: var(--accent); }
  .highlight-num.orange { color: var(--warn); }
  .highlight-title { font-size: 0.95rem; font-weight: 600; color: #fff; margin-bottom: 4px; }
  .highlight-detail { font-size: 0.78rem; color: var(--muted); line-height: 1.5; }

  .criteria-section {
    margin-top: 40px;
    padding-top: 28px;
    border-top: 1px solid var(--border);
  }
  .criteria-top {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    align-items: flex-end;
    gap: 12px;
    margin-bottom: 16px;
  }
  .criteria-header {
    font-size: 0.68rem;
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: var(--muted);
  }
  #filter-count { font-family: ui-monospace, monospace; font-size: 0.72rem; color: var(--muted); }

  .toolbar {
    position: sticky;
    top: 12px;
    z-index: 10;
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    align-items: center;
    padding: 14px 16px;
    margin-bottom: 20px;
    border-radius: 16px;
    border: 1px solid var(--border);
    background: rgba(10, 12, 24, 0.92);
    backdrop-filter: blur(14px);
    box-shadow: 0 12px 40px rgba(0, 0, 0, 0.35);
  }
  .toolbar.is-scrolled { border-color: rgba(110, 231, 168, 0.25); }
  .filter-group { display: flex; flex-wrap: wrap; gap: 6px; }
  .filter-btn, .toolbar select, .toolbar input, .ghost-btn {
    background: rgba(255,255,255,0.04);
    color: var(--text);
    border: 1px solid var(--border);
    border-radius: 999px;
    padding: 7px 14px;
    font-size: 0.76rem;
    cursor: pointer;
    transition: border-color 0.15s, color 0.15s, background 0.15s;
  }
  .filter-btn:hover, .ghost-btn:hover { border-color: rgba(110, 231, 168, 0.4); }
  .filter-btn.active {
    background: rgba(110, 231, 168, 0.15);
    border-color: var(--accent);
    color: var(--accent);
    font-weight: 600;
  }
  .search-wrap { display: flex; align-items: center; gap: 6px; flex: 1; min-width: 180px; max-width: 280px; }
  .search-wrap input { flex: 1; border-radius: 999px; padding-left: 14px; }
  .search-wrap input::placeholder { color: var(--muted); }
  .toolbar select {
    border-radius: 10px;
    padding-right: 28px;
    cursor: pointer;
    color-scheme: dark;
    background-color: rgba(12, 14, 28, 0.95);
    color: var(--text);
  }
  .toolbar select option {
    background-color: #12162a;
    color: var(--text);
  }
  .toolbar-actions { display: flex; gap: 6px; margin-left: auto; }
  .ghost-btn { font-size: 0.72rem; }

  .pillar-group {
    margin-bottom: 20px;
    border-radius: 14px;
    border: 1px solid var(--border);
    overflow: hidden;
    background: rgba(12, 14, 28, 0.45);
  }
  .pillar-header.toggle {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 14px 16px;
    cursor: pointer;
    user-select: none;
    background: rgba(255,255,255,0.02);
    transition: background 0.15s;
  }
  .pillar-header.toggle:hover { background: rgba(110, 231, 168, 0.06); }
  .chevron {
    font-size: 0.7rem;
    color: var(--muted);
    transition: transform 0.2s;
    width: 1em;
  }
  .pillar-group.collapsed .chevron { transform: rotate(-90deg); }
  .pillar-name { font-size: 0.92rem; font-weight: 600; color: #fff; flex: 1; }
  .pillar-mini-bar {
    flex: 0 0 80px;
    height: 4px;
    border-radius: 999px;
    background: rgba(255,255,255,0.08);
    overflow: hidden;
  }
  .pillar-mini-bar span {
    display: block;
    height: 100%;
    background: linear-gradient(90deg, var(--accent), #6366f1);
    border-radius: inherit;
  }
  .pillar-score { font-family: ui-monospace, monospace; font-size: 0.78rem; color: var(--muted); }
  .pillar-body { padding: 0 12px 10px; }
  .pillar-group.collapsed .pillar-body { display: none; }

  .criterion-row {
    display: grid;
    grid-template-columns: 28px 52px 1fr 56px minmax(0, 2fr);
    gap: 10px 14px;
    padding: 10px 8px;
    align-items: start;
    font-size: 0.78rem;
    border-radius: 8px;
    margin-bottom: 2px;
    transition: background 0.12s;
  }
  .criterion-row:hover { background: rgba(255,255,255,0.03); }
  .status-icon { text-align: center; font-size: 0.95rem; line-height: 1.4; }
  .status-icon.pass { color: var(--accent); }
  .status-icon.fail { color: var(--fail); }
  .status-icon.skip { color: var(--skip); }
  .level-tag {
    font-family: ui-monospace, monospace;
    font-size: 0.65rem;
    font-weight: 700;
    padding: 2px 6px;
    border-radius: 4px;
    background: rgba(99, 102, 241, 0.2);
    color: #a5b4fc;
    text-align: center;
  }
  .criterion-name { font-family: ui-monospace, monospace; color: #d4daf0; word-break: break-word; }
  .criterion-score { font-family: ui-monospace, monospace; color: var(--muted); text-align: right; }
  .criterion-evidence { color: var(--muted); line-height: 1.45; word-break: break-word; }
  .criterion-row.changed {
    background: rgba(240, 180, 41, 0.1);
    border-left: 3px solid var(--warn);
    padding-left: 10px;
  }
  .empty-state {
    display: none;
    text-align: center;
    padding: 32px;
    color: var(--muted);
    font-family: ui-monospace, monospace;
    font-size: 0.82rem;
  }
  .empty-state.visible { display: block; }

  .pillar-header {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 14px 16px;
    background: rgba(255,255,255,0.02);
  }

  #tab-remediation .criterion-row {
    grid-template-columns: 28px 1fr 72px minmax(0, 2fr);
  }
  #tab-remediation .criterion-score.impact-high { color: var(--fail); font-weight: 600; }
  #tab-remediation .criterion-score.impact-medium { color: var(--warn); font-weight: 600; }
  #tab-remediation .criterion-score.impact-low { color: var(--muted); }

  footer.report-footer {
    margin-top: 40px;
    padding: 24px 28px;
    border-radius: 16px;
    border: 1px solid var(--border);
    background: var(--surface);
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 16px 24px;
  }
  footer.report-footer .footer-item { display: flex; flex-direction: column; gap: 4px; }
  footer.report-footer .footer-label {
    font-size: 0.62rem;
    text-transform: uppercase;
    letter-spacing: 0.12em;
    color: var(--muted);
  }
  footer.report-footer .footer-value {
    font-family: ui-monospace, monospace;
    font-size: 0.8rem;
    color: #cdd3ef;
    word-break: break-all;
  }
  .brand-line {
    grid-column: 1 / -1;
    padding-top: 12px;
    margin-top: 4px;
    border-top: 1px solid var(--border);
    font-size: 0.72rem;
    color: var(--muted);
    text-align: center;
  }

  @media (max-width: 768px) {
    .hero { grid-template-columns: 1fr; text-align: center; }
    .maturity-badge { margin: 0 auto; }
    .columns { grid-template-columns: 1fr; }
    .level-gates { grid-template-columns: repeat(2, 1fr); }
    .criterion-row { grid-template-columns: 24px 44px 1fr; }
    .criterion-score, .criterion-evidence { grid-column: 2 / -1; }
    .toolbar { flex-direction: column; align-items: stretch; }
    .toolbar-actions { margin-left: 0; justify-content: center; }
  }
  @media print {
    body::before { display: none; }
    .toolbar { position: static; }
    .pillar-group.collapsed .pillar-body { display: block !important; }
  }

  /* Tab Navigation */
  .tab-nav {
    display: flex;
    gap: 0;
    border-bottom: 1px solid var(--border);
    margin-bottom: 32px;
    margin-top: 24px;
  }
  .tab-btn {
    background: none;
    border: none;
    border-bottom: 2px solid transparent;
    color: var(--muted);
    font-family: ui-monospace, monospace;
    font-size: 0.8rem;
    font-weight: 600;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    padding: 12px 24px;
    cursor: pointer;
    transition: color 0.2s, border-color 0.2s;
  }
  .tab-btn:hover { color: var(--text); }
  .tab-btn.active {
    color: #fff;
    border-bottom-color: var(--accent);
  }
  .tab-panel { display: none; }
  .tab-panel.active { display: block; }
</style>
</head>
<body>
<div class="page">

<header class="hero">
  <div>
    <h1>{project_name} <span class="badge">{language}</span></h1>
    <p class="meta">{repo_path}</p>
    <p class="description">{project_description}</p>
  </div>
  <div class="maturity-badge" aria-label="Maturity level {maturity_level}, {maturity_label}">
    <span class="level-num">L{maturity_level}</span>
    <span class="level-sub">{maturity_label}</span>
  </div>
</header>

<div class="stat-grid">
  <div class="stat-card accent"><div class="label">Pass Rate</div><div class="value">{pass_rate}%</div></div>
  <div class="stat-card"><div class="label">Weighted</div><div class="value">{weighted_score}%</div></div>
  <div class="stat-card warn"><div class="label">Evaluated</div><div class="value">{total_applicable}</div></div>
  <div class="stat-card"><div class="label">Skipped</div><div class="value">{criteria_skipped}</div></div>
</div>

<main>
  <section class="level-section">
    <div class="level-bar" role="progressbar" aria-valuenow="{maturity_level}" aria-valuemin="1" aria-valuemax="5" aria-label="Maturity level {maturity_level} of 5">
      <div class="level-segment l1" style="width:{l1_weight}%"></div>
      <div class="level-segment l2" style="width:{l2_weight}%"></div>
      <div class="level-segment l3" style="width:{l3_weight}%"></div>
      <div class="level-segment l4" style="width:{l4_weight}%"></div>
      <div class="level-segment l5" style="width:{l5_weight}%"></div>
    </div>
    <div class="level-labels" aria-hidden="true">
      <div class="level-label" style="width:{l1_weight}%"><span class="pct">{l1_pct}%</span> <span class="name">L1</span></div>
      <div class="level-label" style="width:{l2_weight}%"><span class="pct">{l2_pct}%</span> <span class="name">L2</span></div>
      <div class="level-label" style="width:{l3_weight}%"><span class="pct">{l3_pct}%</span> <span class="name">L3</span></div>
      <div class="level-label" style="width:{l4_weight}%"><span class="pct">{l4_pct}%</span> <span class="name">L4</span></div>
      <div class="level-label" style="width:{l5_weight}%"><span class="pct">{l5_pct}%</span> <span class="name">L5</span></div>
    </div>
    <div class="level-gates">
      <div class="gate-card {l1_gate_class}"><span class="gate-pct">{l1_pct}%</span><span class="gate-name">L1 Functional</span><span class="gate-status">{l1_gate_status}</span></div>
      <div class="gate-card {l2_gate_class}"><span class="gate-pct">{l2_pct}%</span><span class="gate-name">L2 Documented</span><span class="gate-status">{l2_gate_status}</span></div>
      <div class="gate-card {l3_gate_class}"><span class="gate-pct">{l3_pct}%</span><span class="gate-name">L3 Standardized</span><span class="gate-status">{l3_gate_status}</span></div>
      <div class="gate-card {l4_gate_class}"><span class="gate-pct">{l4_pct}%</span><span class="gate-name">L4 Optimized</span><span class="gate-status">{l4_gate_status}</span></div>
      <div class="gate-card {l5_gate_class}"><span class="gate-pct">{l5_pct}%</span><span class="gate-name">L5 Autonomous</span><span class="gate-status">{l5_gate_status}</span></div>
    </div>
  </section>

  <!-- TAB NAVIGATION -->
  <nav class="tab-nav">
    <button class="tab-btn active" data-tab="assessment">Assessment</button>
    <button class="tab-btn" data-tab="remediation">Remediation</button>
  </nav>

  <!-- ASSESSMENT TAB -->
  <div class="tab-panel active" id="tab-assessment">

  <!-- SUMMARY -->
  <section class="summary-section">
    <h2>{summary_headline}<span class="delta-chip">{pass_rate_delta_text}</span></h2>
    <p>{summary_text}</p>
    <div class="columns">
      <div>
        <h3 class="col-header">Strengths</h3>
        <article class="highlight">
          <div class="highlight-num green">01</div>
          <div class="highlight-title">{strength_1_title}</div>
          <div class="highlight-detail">{strength_1_detail}</div>
        </article>
        <!-- Repeat 02, 03 for additional strengths -->
      </div>
      <div>
        <h3 class="col-header">Opportunities</h3>
        <article class="highlight">
          <div class="highlight-num orange">01</div>
          <div class="highlight-title">{opportunity_1_title}</div>
          <div class="highlight-detail">{opportunity_1_detail}</div>
        </article>
        <!-- Repeat 02, 03 for additional opportunities -->
      </div>
    </div>
  </section>

  <section class="criteria-section">
    <div class="criteria-top">
      <h2 class="criteria-header">All Criteria</h2>
      <span id="filter-count">Showing all criteria</span>
    </div>

    <div class="toolbar" id="criteria-toolbar">
      <div class="filter-group">
        <button type="button" class="filter-btn active" data-status="all">Show All</button>
        <button type="button" class="filter-btn" data-status="found">Found</button>
        <button type="button" class="filter-btn" data-status="missing">Missing</button>
        <button type="button" class="filter-btn" data-status="skipped">Skipped</button>
      </div>
      <div class="search-wrap">
        <input id="criteria-search" type="search" placeholder="Search criteria by name…" aria-label="Search criteria">
        <button type="button" class="ghost-btn" id="clear-search" title="Clear search">✕</button>
      </div>
      <select id="level-filter" aria-label="Filter by maturity level">
        <option value="all">All Levels</option>
        <option value="1">Level 1 — Functional</option>
        <option value="2">Level 2 — Documented</option>
        <option value="3">Level 3 — Standardized</option>
        <option value="4">Level 4 — Optimized</option>
        <option value="5">Level 5 — Autonomous</option>
      </select>
      <div class="toolbar-actions">
        <button type="button" class="ghost-btn" id="expand-all">Expand all</button>
        <button type="button" class="ghost-btn" id="collapse-all">Collapse all</button>
      </div>
    </div>

    <div id="criteria-empty" class="empty-state">No criteria match the current filters. Try clearing search or changing status/level.</div>

    <!-- Repeat for each pillar (9 total) -->
    <section class="pillar-group">
      <div class="pillar-header toggle" role="button" tabindex="0" aria-expanded="true">
        <span class="chevron" aria-hidden="true">▼</span>
        <span class="pillar-name">{pillar_name}</span>
        <div class="pillar-mini-bar" aria-hidden="true"><span style="width:{percentage}%"></span></div>
        <span class="pillar-score">{passed}/{total} ({percentage}%)</span>
      </div>
      <div class="pillar-body">
        <!-- Repeat for each criterion in this pillar, alphabetically by name -->
        <div class="criterion-row {changed_class}" data-status="{found|missing|skipped}" data-level="{level}" data-name="{criterion_name}">
          <span class="status-icon {pass|fail|skip}" aria-label="{pass: Passed|fail: Failed|skip: Skipped}">{✓|✗|—}</span>
          <span class="level-tag">L{level}</span>
          <span class="criterion-name">{criterion_name}</span>
          <span class="criterion-score">{1/1|0/1|—/—}</span>
          <span class="criterion-evidence">{evidence_text}</span>
        </div>
      </div>
    </section>
  </section>

  </div><!-- /tab-assessment -->

  <!-- REMEDIATION TAB -->
  <div class="tab-panel" id="tab-remediation">

  <section class="criteria-section" style="margin-top:0;border-top:none;padding-top:0;">
    <h2 class="criteria-header">Remediation Roadmap</h2>
    <p id="remediation-count" style="font-family:ui-monospace,monospace;font-size:0.72rem;color:var(--muted);margin-bottom:20px;">{remediation_count} actionable fixes across missing criteria</p>

    <!-- For each maturity level (L1→L5) with missing criteria: -->
    <section class="pillar-group">
      <div class="pillar-header">
        <span class="pillar-name">Level {N} — {level_label}</span>
        <span class="pillar-score">{missing_count} gaps</span>
      </div>
      <div class="pillar-body">
        <!-- For each missing criterion at this level, ordered by impact (high→medium→low): -->
        <div class="criterion-row">
          <span class="status-icon fail" aria-label="Missing">▸</span>
          <span class="criterion-name">{criterion_name}</span>
          <span class="criterion-score impact-{high|medium|low}">{impact}</span>
          <span class="criterion-evidence">{fix_instruction}</span>
        </div>
      </div>
    </section>
  </section>

  </div><!-- /tab-remediation -->
</main>

<footer class="report-footer">
  <div class="footer-item">
    <span class="footer-label">Assessment date</span>
    <span class="footer-value">{assessment_date}</span>
  </div>
  <div class="footer-item">
    <span class="footer-label">Skill version</span>
    <span class="footer-value">{plugin_version}</span>
  </div>
  <div class="footer-item">
    <span class="footer-label">Criteria evaluated</span>
    <span class="footer-value">{total_applicable}</span>
  </div>
  <div class="footer-item">
    <span class="footer-label">Criteria skipped</span>
    <span class="footer-value">{criteria_skipped}</span>
  </div>
  <div class="footer-item">
    <span class="footer-label">Git SHA</span>
    <span class="footer-value">{git_sha}</span>
  </div>
  <div class="footer-item">
    <span class="footer-label">Assessment duration</span>
    <span class="footer-value">{assessment_duration}</span>
  </div>
  <p class="brand-line">Generated by Agent Fit · agent-readiness assessment</p>
</footer>

<script>
(() => {
  const buttons = Array.from(document.querySelectorAll('.filter-btn'));
  const searchInput = document.getElementById('criteria-search');
  const clearSearch = document.getElementById('clear-search');
  const levelSelect = document.getElementById('level-filter');
  const filterCount = document.getElementById('filter-count');
  const emptyState = document.getElementById('criteria-empty');
  const toolbar = document.getElementById('criteria-toolbar');
  const rows = Array.from(document.querySelectorAll('#tab-assessment .criterion-row'));
  const groups = Array.from(document.querySelectorAll('#tab-assessment .pillar-group'));
  const totalRows = rows.length;
  let statusFilter = 'all';

  const applyFilters = () => {
    let visibleCount = 0;
    rows.forEach((row) => {
      const matchesStatus = statusFilter === 'all' || row.dataset.status === statusFilter;
      const matchesLevel = levelSelect.value === 'all' || row.dataset.level === levelSelect.value;
      const query = (searchInput.value || '').trim().toLowerCase();
      const matchesQuery = !query || (row.dataset.name || '').toLowerCase().includes(query);
      const visible = matchesStatus && matchesLevel && matchesQuery;
      row.style.display = visible ? '' : 'none';
      if (visible) visibleCount += 1;
    });
    groups.forEach((group) => {
      const anyVisible = Array.from(group.querySelectorAll('.criterion-row')).some((row) => row.style.display !== 'none');
      group.style.display = anyVisible ? '' : 'none';
    });
    emptyState.classList.toggle('visible', visibleCount === 0);
    filterCount.textContent = visibleCount === totalRows
      ? `Showing all ${totalRows} criteria`
      : `Showing ${visibleCount} of ${totalRows} criteria`;
  };

  buttons.forEach((btn) => {
    btn.addEventListener('click', () => {
      statusFilter = btn.dataset.status || 'all';
      buttons.forEach((b) => b.classList.toggle('active', b === btn));
      applyFilters();
    });
  });
  searchInput.addEventListener('input', applyFilters);
  clearSearch.addEventListener('click', () => { searchInput.value = ''; searchInput.focus(); applyFilters(); });
  levelSelect.addEventListener('change', applyFilters);

  const setAllCollapsed = (collapsed) => {
    groups.forEach((group) => {
      const header = group.querySelector('.pillar-header.toggle');
      group.classList.toggle('collapsed', collapsed);
      if (header) header.setAttribute('aria-expanded', collapsed ? 'false' : 'true');
    });
  };
  document.getElementById('expand-all').addEventListener('click', () => setAllCollapsed(false));
  document.getElementById('collapse-all').addEventListener('click', () => setAllCollapsed(true));

  groups.forEach((group) => {
    const header = group.querySelector('.pillar-header.toggle');
    if (!header) return;
    const toggle = () => {
      const collapsed = group.classList.toggle('collapsed');
      header.setAttribute('aria-expanded', collapsed ? 'false' : 'true');
    };
    header.addEventListener('click', toggle);
    header.addEventListener('keydown', (event) => {
      if (event.key === 'Enter' || event.key === ' ') {
        event.preventDefault();
        toggle();
      }
    });
  });

  window.addEventListener('scroll', () => {
    toolbar.classList.toggle('is-scrolled', window.scrollY > 80);
  }, { passive: true });

  document.addEventListener('keydown', (event) => {
    if (event.key === '/' && document.activeElement !== searchInput) {
      event.preventDefault();
      searchInput.focus();
    }
  });

  applyFilters();

  document.querySelectorAll('.tab-btn').forEach((btn) => {
    btn.addEventListener('click', () => {
      document.querySelectorAll('.tab-btn').forEach((b) => b.classList.remove('active'));
      document.querySelectorAll('.tab-panel').forEach((p) => p.classList.remove('active'));
      btn.classList.add('active');
      document.getElementById('tab-' + btn.dataset.tab).classList.add('active');
    });
  });
})();
</script>
</body>
</html>

```

**IMPORTANT**: When generating the HTML, replace ALL `{placeholder}` values with actual data from the assessment (including gate card classes/statuses). Do NOT leave any placeholders in the output. You may use `plugins/agentfit/templates/report-template.html` as the canonical structure reference. Generate the complete HTML string and write it to the temp file in a single Bash command.

## Step 5: Report to User

After opening the HTML report, print a brief summary to the console:

```
Agent Fit Report: {project_name}
Pass Rate: {pass_rate}% | Weighted: {weighted_score}% | Level: L{maturity_level} ({maturity_label})
Report: /tmp/agentfit-report-{project_name}.html
Remediation: {remediation_count} actionable fixes in roadmap (see report)

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

Levels:
  L1 Functional:     {l1_pct}% {l1_gate_symbol} (gate: 80%)
  L2 Documented:     {l2_pct}% {l2_gate_symbol} (gate: 80%)
  L3 Standardized:   {l3_pct}% {l3_gate_symbol}
  L4 Optimized:      {l4_pct}% {l4_gate_symbol}
  L5 Autonomous:     {l5_pct}% {l5_gate_symbol}
```

Align level names and percentages in a fixed-width column (pad level labels to ~18 characters) so gate symbols line up in the console. Where symbols are:
- `✓` when that level gate is passed
- `✗` when the level was evaluated but below the 80% gate
- `— (blocked by L{n})` when a previous level gate is not yet passed (omit `(gate: 80%)` on blocked lines)
