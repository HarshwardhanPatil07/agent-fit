# Agent-Fit: Complete Pillar & Criteria Reference Document

## Context

This document is a comprehensive reference of all 14 pillars (9 V1 + 5 V2) and 142 criteria in Agent-Fit. It serves two purposes:
1. A complete inventory with tables and explanations for every criterion
2. A **"Why this criterion?"** rationale section to explain to hackathon judges how we determined each criterion's relevance to agent-readiness

---

## Part 1: Architecture Detection (Run BEFORE Any Criteria)

Before evaluating a single criterion, Agent-Fit detects:

| Detection Step | What It Checks | Why It Matters |
|---|---|---|
| **Project Type** | Monorepo, Library, CLI, Web App, API Service, K8s Operator, Mobile App, IaC | Different architectures have fundamentally different agent-readiness requirements |
| **Language** | Go, Rust, Python, Java/Kotlin, TS/JS, C++, Swift, Elixir | Criteria and fix suggestions must match the language ecosystem |
| **Interface Type (V2)** | REST, GraphQL, gRPC, MCP Server, CLI, Library, Web App | Determines which "agent-as-user" criteria are applicable |

**Key principle**: Criteria that don't apply to the detected architecture are **skipped**, not failed. A CLI tool won't be penalized for missing health checks. A library won't be penalized for missing deployment observability.

---

## Part 2: V1 — Coding Agent Readiness (9 Pillars, 86 Criteria)

> **V1 Question**: *"Can an AI coding agent understand, build, test, and extend this codebase?"*

---

### V1 Pillar 1: Style & Validation (13 criteria)

**Why this pillar?** An AI coding agent needs deterministic guardrails. Without formatters, linters, and type checkers, every AI-generated PR becomes a style roulette — reviewers waste time on formatting instead of logic. Consistent style means AI output blends seamlessly with human code.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `formatter` | config_parsing | Prettier, Black, Ruff format, rustfmt, clang-format, gofumpt, google-java-format, ktlint | Any formatter configured |
| 2 | `lint_config` | config_parsing | ESLint, golangci-lint, Ruff, Clippy, Checkstyle, PMD, detekt, Credo | Linter configured with specific rules |
| 3 | `type_check` | config_parsing | tsconfig.json, mypy, Go/Rust/Java compiler (auto-pass) | Type checking configured |
| 4 | `strict_typing` | config_parsing | TS `strict: true`, mypy `strict`, Go/Rust/Java (auto-pass) | Strict mode enabled |
| 5 | `pre_commit_hooks` | file_existence | .pre-commit-config.yaml, Husky, Lefthook, lint-staged | Any pre-commit hook framework |
| 6 | `naming_consistency` | config + docs | ESLint naming-convention, revive rules, documented conventions | Naming enforced or documented |
| 7 | `large_file_detection` | file_existence | Git LFS `.gitattributes`, pre-commit check-added-large-files | Large file detection configured |
| 8 | `code_modularization` | config_parsing | Go `internal/`, eslint-plugin-boundaries, Nx boundaries, Rust module visibility | Module boundary enforcement exists |
| 9 | `cyclomatic_complexity` | config_parsing | gocyclo, ESLint complexity rule, Ruff C901, Clippy cognitive_complexity | Complexity analysis configured |
| 10 | `dead_code_detection` | config_parsing | knip, vulture, staticcheck unused, Ruff F401/F841, Clippy dead_code | Dead code detection configured |
| 11 | `duplicate_code_detection` | config_parsing | jscpd, PMD CPD, golangci-lint dupl, SonarQube | Duplication scanner configured |
| 12 | `tech_debt_tracking` | ci_workflow | CI scanning TODO/FIXME, godox linter, SonarQube | Tech debt tracked automatically |
| 13 | `n_plus_one_detection` | config_parsing | bullet gem, django-auto-prefetch, nplusone, Prisma warnings | N+1 detection configured (skip if no ORM) |

**Why each criterion matters for agent-readiness:**

- **formatter / lint_config / type_check / strict_typing**: These are the AI agent's "safety net." When an agent generates code, these tools automatically catch style violations, type errors, and lint issues *before* the code reaches a human reviewer. Without them, every AI commit needs manual style review — that's unsustainable.
- **pre_commit_hooks**: Ensures the safety net runs automatically. An agent can run `git commit` and trust that hooks will catch problems before the commit lands.
- **naming_consistency**: Agents pattern-match on existing code. If naming is inconsistent (camelCase mixed with snake_case), the agent will perpetuate the inconsistency, creating tech debt.
- **large_file_detection**: Agents might accidentally commit large generated files (binaries, data dumps). LFS detection prevents bloating the repo.
- **code_modularization**: Agents need clear boundaries to know where to put new code. Without boundaries, an agent might add a utility function in the wrong package.
- **cyclomatic_complexity / dead_code / duplicate_code**: These tell the agent what NOT to do. High complexity limits flag functions that should be refactored, not extended. Dead code detection prevents agents from calling unused functions.
- **tech_debt_tracking**: Gives agents context about known issues. An agent scanning TODOs can prioritize or avoid areas under active rework.
- **n_plus_one_detection**: Database query patterns are one of the hardest things for AI to get right. Automated detection catches the most common performance bug.

---

### V1 Pillar 2: Build System (19 criteria)

**Why this pillar?** If an AI agent can't build and deploy the project, it can't verify its own changes. A one-command setup, pinned dependencies, and fast CI feedback loops are the difference between an agent that ships code and one that submits broken PRs.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `build_cmd_doc` | doc_content | Build/install/run instructions in README, AGENTS.md, Makefile | Build command documented |
| 2 | `deps_pinned` | file_existence | go.sum, package-lock.json, pnpm-lock.yaml, Cargo.lock, uv.lock | Lockfile exists |
| 3 | `vcs_cli_tools` | doc + file | VCS workflow tooling in docs or CI (gh CLI, Prow OWNERS) | VCS tooling documented/automated |
| 4 | `single_command_setup` | doc_content | `make setup`, `docker-compose up`, `npm install` as single command | One-step setup documented |
| 5 | `fast_ci_feedback` | ci_workflow | CI caching (actions/cache, turbo, sccache), manageable matrix size | CI uses caching, no oversized matrix |
| 6 | `deployment_frequency` | git_history | Release tags with dates | Multiple releases/month visible |
| 7 | `release_automation` | ci_workflow | goreleaser, semantic-release, release-please, tag-triggered workflows | Releases automated via CI |
| 8 | `release_notes_automation` | config_parsing | Changelog generators (towncrier, git-cliff, changesets) | Release notes auto-generated |
| 9 | `automated_pr_review` | ci_workflow | CodeRabbit, Copilot Review, Danger.js, Semgrep in PRs | Automated review bots on PRs |
| 10 | `agentic_development` | git_history | Co-authored-by trailers from Claude, Copilot, Devin, cursor-bot | Agent co-authorship in git history |
| 11 | `feature_flag_infrastructure` | config + deps | LaunchDarkly, Statsig, Unleash, custom flags | Feature flag system configured |
| 12 | `build_performance_tracking` | config_parsing | Turborepo remote cache, Nx Cloud, sccache, Gradle build scans | Build times cached/tracked |
| 13 | `unused_deps_detection` | ci + config | depcheck, `go mod tidy`, cargo-udeps, deptry, knip | Unused deps detected automatically |
| 14 | `monorepo_tooling` | config_parsing | Lerna, Nx, Turbo, Bazel, Cargo workspaces, pnpm-workspace | Monorepo tooling exists (skip if not monorepo) |
| 15 | `dead_feature_flag_detection` | config_parsing | Stale flag reports, lifecycle tooling | Dead flag detection (skip if no flags) |
| 16 | `heavy_dependency_detection` | config_parsing | bundlewatch, size-limit, webpack-bundle-analyzer | Bundle size monitoring (skip if not frontend) |
| 17 | `progressive_rollout` | config_parsing | Canary deploy, percentage-based rollout, blue-green | Progressive rollout configured |
| 18 | `rollback_automation` | config + ci | Automated rollback on error, one-click rollback | Automated rollback exists |
| 19 | `version_drift_detection` | config_parsing | syncpack, manypkg, version consistency checks | Version drift detected (skip if not monorepo) |

**Why each criterion matters:**

- **build_cmd_doc / single_command_setup**: An AI agent's first action is to build the project. If it can't find a build command, it's stuck. A single command means the agent doesn't need to figure out multi-step procedures.
- **deps_pinned**: Without a lockfile, `npm install` on Tuesday gives different packages than on Wednesday. An agent debugging a build failure caused by version drift is wasting everyone's time.
- **fast_ci_feedback**: Agents iterate fast. If CI takes 45 minutes, the agent's feedback loop is broken. Caching keeps CI fast enough for agents to iterate productively.
- **release_automation / release_notes_automation**: Agents can trigger releases, but only if the release process is automated. Manual release steps are a human bottleneck.
- **automated_pr_review / agentic_development**: These measure whether the repo is *already* agent-friendly in practice. If other agents are already contributing, the infrastructure is proven.
- **feature_flag_infrastructure**: Agents can ship behind flags, reducing risk. Without flags, every agent change is all-or-nothing.

---

### V1 Pillar 3: Testing (8 criteria)

**Why this pillar?** Tests are how an AI agent verifies its own work. Without tests, an agent can't tell if its changes broke something. Without test isolation, parallel agent runs interfere with each other.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `unit_tests_exist` | file_existence | Test files matching language conventions | Test files exist |
| 2 | `unit_tests_runnable` | doc_content | `go test`, `pytest`, `jest`, `cargo test` documented | Test command documented |
| 3 | `integration_tests_exist` | file_existence | Playwright, Cypress, e2e/, integration/ directories | Integration/E2E tests exist |
| 4 | `test_coverage_thresholds` | config_parsing | Codecov targets, Jest coverageThreshold, pytest-cov minimum | Coverage minimum enforced |
| 5 | `test_naming_conventions` | code_search | Language-idiomatic test naming patterns | Consistent naming used |
| 6 | `test_isolation` | config_parsing | t.Parallel(), pytest-xdist, Jest workers, -race flag | Test isolation configured |
| 7 | `flaky_test_detection` | config + ci | pytest-rerunfailures, retry-on-failure, quarantine | Flaky tests detected/quarantined |
| 8 | `test_performance_tracking` | config_parsing | --durations flags, gotestsum timing, test analytics | Test execution times monitored |

**Why each criterion matters:**

- **unit_tests_exist / unit_tests_runnable**: Foundational. An agent can run `make test` after every change to check for regressions. No tests = the agent is flying blind.
- **integration_tests_exist**: Unit tests verify functions; integration tests verify the system works end-to-end. Agents need both.
- **test_coverage_thresholds**: Prevents agents from reducing coverage. If the threshold is 80%, the agent knows it must add tests for new code.
- **test_naming_conventions**: Agents read test names to understand what behavior is covered. Consistent naming makes this reliable.
- **test_isolation**: Multiple agents (or agent + human) may run tests simultaneously. Without isolation, tests interfere with each other and produce false failures.
- **flaky_test_detection**: Flaky tests are an agent's worst enemy. The agent changes nothing, tests fail, and the agent wastes cycles debugging a phantom issue.
- **test_performance_tracking**: Slow tests slow down the agent's iteration cycle.

---

### V1 Pillar 4: Documentation (10 criteria)

**Why this pillar?** AI agents learn by reading. The quality, structure, and freshness of documentation directly determines how well an agent understands the codebase. An `AGENTS.md` file is the single highest-ROI documentation investment for agent-readiness.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `readme` | file + doc | README.md with description, install, usage, API docs | README with ≥2 key sections |
| 2 | `agents_md` | file_existence | AGENTS.md or CLAUDE.md with agent instructions | Agent instructions file exists |
| 3 | `agents_md_validation` | ci_workflow | CI validates agent instructions | CI validates agent instructions |
| 4 | `documentation_freshness` | git_history | Key docs modified within 180 days | Docs are fresh |
| 5 | `automated_doc_generation` | ci + config | Sphinx, MkDocs, Typedoc, godoc, `make docs` | Docs auto-generated from code |
| 6 | `api_schema_docs` | file_existence | OpenAPI, .proto files, GraphQL schema | API schemas documented |
| 7 | `service_flow_documented` | file_existence | PlantUML, Mermaid, architecture diagrams | Architecture diagrams exist |
| 8 | `skills` | file_existence | .claude/skills/ or .factory/skills/ directories | Reusable agent skills defined |
| 9 | `changelog_maintained` | file + doc | CHANGELOG.md with structured entries | Maintained changelog exists |
| 10 | `doc_examples_tested` | config + code | Doctest, Go testable examples, Rust doc-tests | Doc examples verified by tests |

**Why each criterion matters:**

- **readme**: The agent's entry point. Without a README, the agent has to reverse-engineer the project's purpose.
- **agents_md / skills**: The most direct signal of agent-readiness. AGENTS.md tells agents what commands to run, what conventions to follow, and what to avoid. Skills provide reusable workflows.
- **agents_md_validation**: Ensures AGENTS.md doesn't go stale. Stale agent instructions are worse than none — they actively mislead.
- **documentation_freshness**: 6-month-old docs may describe a codebase that no longer exists. Agents need current information.
- **api_schema_docs**: Structured schemas (OpenAPI, protobuf) are machine-readable. Agents can parse them to understand endpoints automatically.
- **doc_examples_tested**: Untested examples rot. If an agent follows a code example from the docs and it doesn't work, trust is broken.

---

### V1 Pillar 5: Development Environment (5 criteria)

**Why this pillar?** An AI agent needs to set up a development environment autonomously. Devcontainers, env templates, and containerized local services let an agent go from zero to running in a single command.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `devcontainer` | file_existence | .devcontainer/, flake.nix, shell.nix | Reproducible dev env configured |
| 2 | `devcontainer_runnable` | config_parsing | Valid base image, features, or Nix inputs | Dev env appears buildable (skip if no config) |
| 3 | `env_template` | file_existence | .env.example, .tool-versions, .mise.toml, .nvmrc, Go version in go.mod | Env template or version pinning exists |
| 4 | `local_services_setup` | file_existence | docker-compose.yml with service definitions | Local services containerized |
| 5 | `database_schema` | file_existence | Migration files, schema.prisma, SQLAlchemy models | Schema management exists (skip if no DB) |

**Why each criterion matters:**

- **devcontainer / devcontainer_runnable**: The gold standard. An agent opens a devcontainer and has every tool pre-installed. No "it works on my machine" problems.
- **env_template**: Agents need to know which environment variables are required. `.env.example` is a machine-readable specification of dependencies.
- **local_services_setup**: If the app needs Postgres and Redis, docker-compose spins them up. Without it, the agent has to figure out external service setup.
- **database_schema**: Migration files tell the agent how the database evolves. Without them, schema changes are manual and error-prone.

---

### V1 Pillar 6: Debugging & Observability (11 criteria)

**Why this pillar?** When an AI agent's changes cause issues in production, observability tools help diagnose problems. Structured logging, tracing, and metrics make agent-generated code debuggable.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `structured_logging` | dependency_check | zap, logrus, structlog, winston, pino, tracing crate | Structured logging library used |
| 2 | `distributed_tracing` | dependency_check | OpenTelemetry, Jaeger, X-Request-ID, dd-trace | Distributed tracing instrumented |
| 3 | `metrics_collection` | dependency_check | Prometheus, Datadog StatsD, OpenTelemetry metrics | Metrics collected |
| 4 | `alerting_configured` | config_parsing | Alertmanager, PagerDuty, OpsGenie rules | Alerting configured |
| 5 | `deployment_observability` | config + doc | Grafana dashboards, monitoring docs, deploy notifications | Deployment monitoring exists |
| 6 | `error_tracking_contextualized` | dependency_check | Sentry, Bugsnag, Rollbar with release tracking | Error tracking service configured |
| 7 | `health_checks` | code_search | `/health`, `/healthz` endpoint definitions | Health check endpoints exist |
| 8 | `profiling_instrumentation` | code + deps | pprof, py-spy, Pyroscope, tracy | Profiling infrastructure exists |
| 9 | `code_quality_metrics` | ci_workflow | Codecov, SonarQube, CodeClimate, Coveralls | Code quality/coverage tracked in CI |
| 10 | `circuit_breakers` | dependency_check | gobreaker, resilience4j, hystrix, cockatiel | Resilience patterns implemented |
| 11 | `runbooks_documented` | file_existence | runbooks/ directory, INCIDENT_RESPONSE.md | Runbooks exist |

**Why each criterion matters:**

- **structured_logging**: Agents should log in structured JSON so logs are searchable. Unstructured `console.log` is useless at scale.
- **distributed_tracing / metrics_collection**: When an agent's code change causes a latency spike, traces and metrics pinpoint the cause without manual debugging.
- **health_checks**: Agents deploying services need endpoints to verify the service is running correctly after deployment.
- **circuit_breakers**: Agents adding external API calls need resilience patterns. Without circuit breakers, one failing dependency cascades to everything.

---

### V1 Pillar 7: Security & Governance (13 criteria)

**Why this pillar?** AI agents can accidentally introduce security vulnerabilities — hardcoded secrets, missing input validation, vulnerable dependencies. Automated security scanning catches these before they reach production.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `branch_protection` | api_check | GitHub branch protection rules/rulesets | Branch protection exists (skip if no gh) |
| 2 | `codeowners` | file_existence | CODEOWNERS, Prow OWNERS files | Code ownership file exists |
| 3 | `secret_scanning` | api_check | GitHub secret scanning enabled | Secret scanning enabled (skip if no gh) |
| 4 | `secrets_management` | config + code | Vault, AWS Secrets Manager, SOPS, GitHub Actions secrets | Secrets use a vault/manager |
| 5 | `dependency_update_automation` | file_existence | Dependabot, Renovate configured | Dependency bot configured |
| 6 | `gitignore_comprehensive` | config_parsing | .gitignore covers language-appropriate patterns | Gitignore covers build artifacts + IDE |
| 7 | `automated_security_review` | ci_workflow | CodeQL, Snyk, Trivy, Semgrep, gosec in CI | Security scanning in CI |
| 8 | `log_scrubbing` | code_search | Redaction functions, password masking, sanitization | Sensitive data redacted from logs |
| 9 | `pii_handling` | code_search | Data classification, GDPR controls, PII detection | PII classified and handled |
| 10 | `dast_scanning` | ci_workflow | OWASP ZAP, Burp Suite, Nuclei in CI | DAST in CI (skip if library/CLI) |
| 11 | `privacy_compliance` | config_parsing | Data retention, consent management, GDPR/CCPA | Privacy compliance enforced |
| 12 | `container_image_scanning` | ci_workflow | Trivy, Snyk Container, Grype, docker scout | Container scanning in CI |
| 13 | `sbom_generation` | ci + config | syft, cyclonedx-bom, spdx-sbom-generator | SBOM generated in build/release |

**Why each criterion matters:**

- **branch_protection / codeowners**: Prevents agents from merging directly to main without review. Codeowners ensures the right humans review agent-generated changes in critical areas.
- **secret_scanning / secrets_management**: Agents might accidentally hardcode API keys. Secret scanning catches this; vault integration means agents never see raw secrets.
- **dependency_update_automation**: Agents add dependencies. Dependabot/Renovate automatically flags when those dependencies have known vulnerabilities.
- **automated_security_review**: SAST tools (CodeQL, Semgrep) catch agent-introduced vulnerabilities — SQL injection, XSS, path traversal — in the PR, not in production.
- **gitignore_comprehensive**: Prevents agents from accidentally committing `.env` files, build artifacts, or IDE configs.

---

### V1 Pillar 8: Task Discovery (4 criteria)

**Why this pillar?** AI agents need to find and understand work items. Well-structured issues with labels give agents the context to pick up tasks autonomously.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `issue_templates` | file_existence | .github/ISSUE_TEMPLATE/ with templates | Structured issue templates exist |
| 2 | `issue_labeling_system` | api_check | Custom labels beyond GitHub defaults | Custom label taxonomy (skip if no gh) |
| 3 | `backlog_health` | api_check | >70% of issues have labels and meaningful titles | Backlog well-maintained (skip if no gh) |
| 4 | `pr_templates` | file_existence | .github/pull_request_template.md | PR template with structured sections |

**Why each criterion matters:**

- **issue_templates**: Structured templates ensure agents get consistent information: steps to reproduce, expected behavior, acceptance criteria. Unstructured issues leave agents guessing.
- **issue_labeling_system**: Labels like `priority/high`, `area/auth`, `kind/bug` let agents filter and prioritize work.
- **backlog_health**: A messy backlog with unlabeled, vague issues is unusable by agents. Well-maintained backlogs are machine-readable work queues.
- **pr_templates**: Templates give agents a checklist for what to include in PRs (description, test plan, screenshots), improving PR quality.

---

### V1 Pillar 9: Product & Analytics (3 criteria)

**Why this pillar?** Agents making product changes need to know if those changes improve user outcomes. Analytics and experiment infrastructure close the feedback loop.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `product_analytics_instrumentation` | dependency_check | Mixpanel, Amplitude, PostHog, GA4, Heap | Analytics instrumented (skip if server/lib) |
| 2 | `error_to_insight_pipeline` | config_parsing | Sentry-GitHub integration, auto issue creation from errors | Errors auto-create tickets |
| 3 | `experiment_infrastructure` | config_parsing | A/B testing, feature flags with metrics, experiment platform | Experiment infrastructure exists |

**Why each criterion matters:**

- **product_analytics**: If an agent ships a UI change, analytics show whether users actually use it. Without data, agent-generated features can't be validated.
- **error_to_insight_pipeline**: When agent code causes errors, automated ticketing ensures someone (human or agent) picks up the fix.
- **experiment_infrastructure**: Agents can ship behind experiments, measuring impact before full rollout. This reduces the risk of agent-generated changes.

---

## Part 3: V2 — Agent-as-User Readiness (5 Pillars, 56 Criteria)

> **V2 Question**: *"Can an AI agent consume this software as an end-user or customer — call its APIs, authenticate, discover its capabilities, and handle errors gracefully?"*

This is the other side of the coin. V1 asks "can an agent build this?" V2 asks "can an agent USE this?"

---

### V2 Pillar 1: Machine-Readable Interfaces (12 criteria)

**Why this pillar?** Agents interact with software through programmatic interfaces, not GUIs. If your software doesn't expose machine-readable interfaces (APIs, CLI with JSON output, MCP tools), agents simply cannot use it.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `v2_rest_api_exists` | code_search | HTTP route handlers (Express, FastAPI, Gin, etc.) | REST API endpoints exist |
| 2 | `v2_openapi_spec` | file + deps | openapi.yaml/json, swagger files, auto-gen tools (FastAPI, nestjs/swagger) | OpenAPI spec exists or auto-generated |
| 3 | `v2_mcp_server` | file + config + code | MCP SDK imports, server.tool() calls, .well-known/mcp.json | MCP server with ≥1 tool definition |
| 4 | `v2_cli_structured_output` | code + doc | `--output json`, `--format json`, `--json` flag | CLI has JSON output option (skip if no CLI) |
| 5 | `v2_grpc_protobuf` | file + deps | .proto files, buf.yaml, gRPC dependencies | gRPC/protobuf exists (skip if different paradigm) |
| 6 | `v2_graphql_schema` | file + deps | *.graphql files, apollo-server, gqlgen, strawberry | GraphQL schema exists (skip if different paradigm) |
| 7 | `v2_webhook_events` | code + doc | Webhook handlers, event systems, subscriber registration | Webhook/event system for external consumers |
| 8 | `v2_api_versioning` | code + config | `/v1/`, `/v2/` route prefixes, Accept-Version header | API versioning implemented (skip if no API) |
| 9 | `v2_sdk_client_libraries` | file + doc | sdk/, client/ directories, auto-generated clients | SDK/client library provided (skip if IS a library) |
| 10 | `v2_batch_bulk_endpoints` | code_search | /batch, /bulk route handlers, batch_create, bulk_update | Batch/bulk endpoints exist (skip if no API) |
| 11 | `v2_webmcp_support` | code + deps | WebMCP implementation, webmcp in dependencies | WebMCP support (skip if no web component) |
| 12 | `v2_sse_streaming_support` | code + deps | text/event-stream endpoints, WebSocket streaming | SSE/streaming endpoints (skip if no server) |

**Why each criterion matters:**

- **v2_rest_api_exists / v2_openapi_spec**: REST is the lingua franca of agent-to-software communication. An OpenAPI spec lets agents auto-generate client code — no guessing about endpoints, parameters, or response shapes.
- **v2_mcp_server**: MCP (Model Context Protocol) is the emerging standard for AI agent integration. An MCP server lets any MCP-compatible agent (Claude, etc.) directly invoke your software's capabilities as tools.
- **v2_cli_structured_output**: Agents can't parse human-readable table output. `--json` output means agents can reliably extract data from CLI tools.
- **v2_api_versioning**: Agents break when APIs change under them. Versioned APIs give agents stability guarantees.
- **v2_sdk_client_libraries**: SDKs lower the barrier for agents to integrate. Instead of constructing HTTP requests, agents call typed methods.
- **v2_batch_bulk_endpoints**: Agents often need to operate on many items at once. Without batch endpoints, agents make N sequential requests — slow and rate-limit-risky.
- **v2_sse_streaming_support**: Agents processing long-running operations need streaming to avoid timeouts and get incremental results.

---

### V2 Pillar 2: Authentication & Access (12 criteria)

**Why this pillar?** Agents are non-human users. If your auth system requires a human (CAPTCHA, browser-based OAuth, manual signup), agents can't authenticate. Machine-to-machine auth is the unlock.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `v2_api_key_auth` | code + config | API key validation middleware, X-API-Key, Bearer token | API key auth exists (skip if no auth) |
| 2 | `v2_no_captcha_on_api` | deps + code | CAPTCHA not applied to API paths | API free from CAPTCHA (auto-pass if no CAPTCHA) |
| 3 | `v2_oauth2_m2m` | code + deps | client_credentials grant, M2M token endpoint | M2M OAuth2 flow supported (skip if no auth) |
| 4 | `v2_service_account_support` | code + doc | Service/bot account types, non-interactive user model | Service accounts supported (skip if no users) |
| 5 | `v2_auth_docs_for_machines` | doc_content | Auth docs with curl/SDK code examples | Auth documented for programmatic use |
| 6 | `v2_scoped_permissions` | code_search | RBAC, permission/scope definitions, OAuth scopes | Scoped permissions exist (skip if no auth) |
| 7 | `v2_token_refresh` | code_search | Refresh token generation, /refresh endpoints | Token refresh exists (skip if no auth) |
| 8 | `v2_programmatic_key_management` | code_search | API endpoints for key create/rotate/revoke | Keys manageable via API (skip if no key system) |
| 9 | `v2_oauth_discovery_metadata` | file + code | .well-known/oauth-authorization-server, OIDC discovery | OAuth discovery metadata served |
| 10 | `v2_sandbox_test_environment` | doc + config | Sandbox/test mode, documented test endpoints | Sandbox env documented (skip if lib/CLI) |
| 11 | `v2_pkce_support` | code_search | PKCE (RFC 7636), code_verifier, code_challenge, S256 | PKCE implemented (skip if no OAuth) |
| 12 | `v2_web_bot_auth` | code + file | HTTP message signatures (RFC 9421) | Bot auth via signatures (skip if no server) |

**Why each criterion matters:**

- **v2_api_key_auth / v2_oauth2_m2m**: These are the two primary ways agents authenticate. API keys for simple cases, OAuth2 client_credentials for enterprise. Without either, agents can't access your software.
- **v2_no_captcha_on_api**: CAPTCHA is literally an "anti-robot test." If your API endpoints have CAPTCHA, agents are blocked by design.
- **v2_service_account_support**: Agents aren't humans. They need non-interactive accounts with appropriate permissions, not shared human credentials.
- **v2_scoped_permissions**: Principle of least privilege for agents. An agent should only have access to the endpoints it needs, not full admin.
- **v2_token_refresh**: Long-running agents need to refresh tokens without human intervention.
- **v2_programmatic_key_management**: Agents should be able to rotate their own keys programmatically, not require a human to go to a settings page.
- **v2_sandbox_test_environment**: Agents need a safe place to test integrations without affecting production data.

---

### V2 Pillar 3: Agent Documentation (12 criteria)

**Why this pillar?** Agents learn to use your software by reading documentation. Machine-parseable docs (structured README, error code tables, API references) are dramatically more useful to agents than prose-heavy wikis.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `v2_agents_md` | file + doc | AGENTS.md/CLAUDE.md with interface docs, API endpoints, auth | Agent instruction file with interface docs |
| 2 | `v2_getting_started_with_code` | doc_content | Getting-started with curl/fetch/SDK examples in <5 steps | Getting started with code examples |
| 3 | `v2_structured_readme` | doc_content | README with ≥3 of: H2 headers, code blocks, tables, lists | Machine-parseable README |
| 4 | `v2_structured_api_reference` | file + doc | docs/api/, Swagger UI, Redoc, TypeDoc, GoDoc | Machine-parseable API reference |
| 5 | `v2_error_codes_documented` | doc + code | Error code → meaning mapping, error constants | Error codes documented |
| 6 | `v2_auth_quickstart` | doc_content | Step-by-step auth with code: get creds → configure → first request | Auth quickstart with code (skip if no auth) |
| 7 | `v2_rate_limits_documented` | doc + code | Rate limit thresholds, X-RateLimit header docs | Rate limits documented (skip if none) |
| 8 | `v2_llms_txt` | file_existence | llms.txt at root, public/, or static/ | llms.txt exists |
| 9 | `v2_llms_full_txt` | file_existence | llms-full.txt at root, public/, or static/ | llms-full.txt exists |
| 10 | `v2_sdk_documentation` | doc + file | SDK READMEs in sdk/client/ with usage examples | SDK docs exist (skip if no SDK) |
| 11 | `v2_changelog_api_migration` | file + doc | CHANGELOG with API entries, MIGRATION.md | API changelog/migration guide (skip if no API) |
| 12 | `v2_markdown_content_negotiation` | code_search | `Accept: text/markdown` content negotiation | Server responds with Markdown (skip if no server) |

**Why each criterion matters:**

- **v2_agents_md**: Tells agents HOW to interact with this software — which endpoints to call, how to authenticate, what capabilities exist.
- **v2_getting_started_with_code**: Agents need code examples, not prose descriptions. "Run `curl -X POST /api/users`" is actionable; "Create a user via the API" is not.
- **v2_structured_api_reference**: OpenAPI/Swagger UI is machine-parseable. An agent can extract every endpoint, parameter, and response type automatically.
- **v2_error_codes_documented**: When an agent gets `ERR_RATE_LIMIT_EXCEEDED`, it needs to know what to do. A documented error code table maps codes to recovery actions.
- **v2_llms_txt / v2_llms_full_txt**: The emerging standard for providing LLM-optimized documentation. `llms.txt` is a concise summary; `llms-full.txt` is the complete reference.
- **v2_rate_limits_documented**: Agents must know rate limits to avoid hitting them. Without docs, agents discover limits by getting 429 errors.

---

### V2 Pillar 4: Discoverability (10 criteria)

**Why this pillar?** Before an agent can use your software, it needs to FIND it and understand what it offers. Discoverability is about making your software visible and comprehensible to AI agents through standard protocols and manifests.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `v2_robots_txt_allows_agents` | file + config | robots.txt with Allow for GPTBot, ClaudeBot, Anthropic | AI bots allowed (skip if CLI/lib) |
| 2 | `v2_plugin_skill_manifest` | file_existence | .claude-plugin/plugin.json, .well-known/ai-plugin.json | AI plugin/skill manifest exists |
| 3 | `v2_mcp_server_card` | file + config | .well-known/mcp.json with name, description, capabilities | MCP server card with metadata (skip if no MCP) |
| 4 | `v2_api_catalog_manifest` | file + doc | .well-known/api-catalog (RFC 9727), capabilities.json | Machine-readable API catalog |
| 5 | `v2_sitemap_structured_index` | file_existence | sitemap.xml, docs/index.json | Sitemap or doc index (skip if no web) |
| 6 | `v2_schema_structured_data` | code + file | JSON-LD, Schema.org markup, JSON Schema | Structured data markup (skip if no web) |
| 7 | `v2_agent_skills_index` | file_existence | .well-known/agent-skills/index.json | Agent skills index (skip if no web/API) |
| 8 | `v2_link_headers_discovery` | code_search | Link response headers (RFC 8288) to API catalog/docs | Link headers configured (skip if no server) |
| 9 | `v2_content_signals` | file + config | Content-Signal directives in robots.txt | Content-Signal directives (skip if no web) |
| 10 | `v2_a2a_agent_card` | file + code | .well-known/agent-card.json (A2A protocol) | A2A Agent Card defined (skip if not agent-capable) |

**Why each criterion matters:**

- **v2_robots_txt_allows_agents**: If your `robots.txt` blocks `ClaudeBot` or `GPTBot`, AI agents can't even read your public documentation. This is the most basic gate.
- **v2_plugin_skill_manifest**: Plugin manifests (like `.well-known/ai-plugin.json`) let agent platforms automatically discover and integrate with your software.
- **v2_mcp_server_card**: The MCP equivalent of an OpenAPI spec — describes what your MCP server can do so agents can decide whether to connect.
- **v2_api_catalog_manifest**: RFC 9727 defines a standard way to advertise all APIs a service provides. Agents can crawl this to discover capabilities.
- **v2_a2a_agent_card**: Google's Agent-to-Agent protocol. An agent card at `.well-known/agent-card.json` lets other agents discover and communicate with your agent.
- **v2_content_signals**: Emerging standard for declaring how AI agents may use your content (training, retrieval, etc.).

---

### V2 Pillar 5: Agent Experience — AX (10 criteria)

**Why this pillar?** Even if agents CAN access your API, bad API design makes them fail. Structured errors, cursor pagination, idempotency, and correlation IDs are the difference between an agent that reliably integrates and one that crashes on edge cases.

| # | Criterion | Signal Type | What It Checks | Pass Condition |
|---|---|---|---|---|
| 1 | `v2_structured_error_responses` | code_search | RFC 7807 Problem Details, JSON error envelope | Structured JSON errors (skip if no API) |
| 2 | `v2_health_check_endpoint` | code_search | /health, /healthz, /ready endpoints | Health check exists (skip if CLI/lib) |
| 3 | `v2_cursor_pagination` | code_search | cursor, next_cursor, page_token (NOT offset-based) | Cursor pagination (skip if no list endpoints) |
| 4 | `v2_rate_limit_headers` | code_search | X-RateLimit-Limit, Remaining, Reset headers | Rate limit headers sent (skip if no rate limiting) |
| 5 | `v2_consistent_response_envelope` | code_search | `{ "data": ..., "meta": ... }` wrapper pattern | Consistent response format (skip if no API) |
| 6 | `v2_idempotency_support` | code_search | Idempotency-Key header, request deduplication | Idempotency supported (skip if no write endpoints) |
| 7 | `v2_request_correlation_id` | code_search | X-Request-ID / X-Correlation-ID generation | Correlation IDs propagated (skip if no API) |
| 8 | `v2_retry_after_support` | code_search | Retry-After header in 429/503 responses | Retry-After sent (skip if no rate limiting) |
| 9 | `v2_async_operation_patterns` | code_search | 202 Accepted + Location, polling /status/{id} endpoints | Async operation patterns (skip if all sync) |
| 10 | `v2_cors_configured` | code + config | CORS middleware, Access-Control-Allow-Origin | CORS configured (skip if CLI/lib/no server) |

**Why each criterion matters:**

- **v2_structured_error_responses**: An HTML 500 page is useless to an agent. A JSON `{ "error": "rate_limit_exceeded", "retry_after": 30 }` is actionable. Agents parse errors to decide what to do next.
- **v2_cursor_pagination**: Offset pagination breaks under concurrent writes (items shift). Cursor pagination is stable and agent-friendly — the agent follows `next_cursor` without tracking page numbers.
- **v2_idempotency_support**: Network failures happen. If an agent retries a POST request, idempotency keys prevent duplicate orders, payments, or records. This is critical for reliability.
- **v2_rate_limit_headers**: `X-RateLimit-Remaining: 5` tells the agent to slow down. Without it, the agent discovers the limit by getting blocked.
- **v2_retry_after_support**: When rate-limited, `Retry-After: 30` tells the agent exactly when to try again. Without it, agents either wait too long or hammer the API.
- **v2_request_correlation_id**: When debugging agent interactions, correlation IDs link the agent's request to server-side logs. Without them, tracing agent issues across systems is impossible.
- **v2_consistent_response_envelope**: A consistent `{ "data": [...], "meta": { "total": 100 } }` pattern means agents can write one parser for all endpoints. Inconsistent shapes require per-endpoint parsing.
- **v2_async_operation_patterns**: Long-running operations (image processing, data exports) need 202 + polling. Without this, agents hit timeouts on slow operations.
- **v2_cors_configured**: Browser-based agents need CORS. Without it, cross-origin requests from agent UIs are blocked.

---

## Part 4: How to Explain to Judges — "Why These Criteria?"

### The Core Argument

> **"We didn't invent these criteria arbitrarily. Every criterion answers one testable question: does the absence of this signal cause an AI agent to fail, slow down, or produce worse results?"**

### The Framework for Justification

For every criterion, we applied this decision test:

1. **Does an agent need this to function?** (L1 criteria — basic build, test, type check)
2. **Does this reduce agent errors?** (L2-L3 — linting, coverage, security scanning)
3. **Does this make agents faster?** (L3-L4 — CI caching, test isolation, observability)
4. **Does this make agents autonomous?** (L4-L5 — experiment infra, error-to-insight pipeline)

### The Two-Sided Assessment Pitch

> "Most tools only check if AI can code IN your repo. We also check if AI can USE your software as a customer. That's the V1/V2 split — two independent scores, because a codebase can be easy to develop on but impossible for agents to consume, or vice versa."

### Architecture Awareness Pitch

> "A CLI tool doesn't need health checks. A library doesn't need product analytics. A Kubernetes operator uses `envtest`, not `docker-compose`. We detect your project type and skip criteria that don't apply — no false negatives, no irrelevant noise."

### Maturity Level Pitch

> "We don't just give a pass/fail score. We map criteria to 5 maturity levels (L1-L5) with an 80% gate at each level. This tells you not just WHERE you are, but WHAT to fix next. Fix all L1 gaps before worrying about L3. It's a roadmap, not just a grade."

---

## Part 5: Summary Statistics

| Dimension | Pillars | Criteria | Maturity Levels |
|---|---|---|---|
| V1 — Coding Agent Readiness | 9 | 86 | L1 Functional → L5 Autonomous |
| V2 — Agent-as-User Readiness | 5 | 56 | L1 Accessible → L4 Autonomous |
| **Total** | **14** | **142** | **Dual-track scoring** |

### V1 Maturity Level Distribution

| Level | Name | # Criteria | Focus |
|---|---|---|---|
| L1 | Functional | 13 | Basic: formatter, linter, types, tests, README, deps, gitignore |
| L2 | Documented | 16 | Process: pre-commit, AGENTS.md, devcontainer, branch protection, CODEOWNERS |
| L3 | Standardized | 31 | Automation: coverage, security scanning, CI caching, release automation |
| L4 | Optimized | 19 | Excellence: profiling, alerting, circuit breakers, DAST, agentic dev |
| L5 | Autonomous | 7 | Self-improving: experiments, error-to-insight, progressive rollout |

### V2 Maturity Level Distribution

| Level | Name | # Criteria | Focus |
|---|---|---|---|
| L1 | Accessible | 8 | Basic: REST API exists, API key auth, no CAPTCHA, structured errors |
| L2 | Documented | 16 | Process: OpenAPI, MCP, structured docs, cursor pagination, CORS |
| L3 | Optimized | 22 | Automation: webhooks, versioning, llms.txt, idempotency, A2A card |
| L4 | Autonomous | 10 | Excellence: SDK generation, key management, retry-after, WebMCP |

---

## Verification

This document was extracted from `plugins/agentfit/commands/agentfit.md` — the single source of truth for all criteria definitions. All 142 criteria, their signal types, pass conditions, and skip rules are documented above.
