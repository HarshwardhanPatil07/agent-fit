# Pillar 2: Build System

Scan for 19 signals covering build automation, CI/CD, dependency management, and release pipelines.

---

## Signal: `build_cmd_documented`

**Purpose:** Agents need to know how to build the project without guessing.

**Detection:**

1. Search documentation files for build/run commands:
   - `CLAUDE.md`, `AGENTS.md` — grep for `build`, `make`, `run`, `compile`, `install`
   - `README.md` — grep for code blocks containing build commands
   - `CONTRIBUTING.md` — grep for build/setup sections
   - `Makefile`, `GNUmakefile`, `makefile` — existence implies documented build commands; check for `help` target
   - `justfile` — Just command runner
   - `Taskfile.yml`, `Taskfile.yaml` — Task runner
2. Search `package.json` for `scripts` section (especially `build`, `dev`, `start`, `test`)
3. Search for `Dockerfile`, `docker-compose.yml` with build targets
4. Search for build tool configs: `CMakeLists.txt`, `BUILD`, `BUILD.bazel`, `build.gradle`, `pom.xml`, `meson.build`

**Output fields:** `found`, `tools`, `config_files`, `documented_commands` (list of build commands found), `evidence`

---

## Signal: `deps_pinned`

**Purpose:** Pinned dependencies ensure reproducible builds.

**Detection:**

1. Search for lockfiles:
   - `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb`, `bun.lock`
   - `go.sum`
   - `Cargo.lock`
   - `poetry.lock`, `Pipfile.lock`, `uv.lock`, `pdm.lock`
   - `Gemfile.lock`
   - `composer.lock`
   - `pubspec.lock`
   - `flake.lock`
   - `Package.resolved` (Swift)
2. Note: For Rust libraries, `Cargo.lock` is often gitignored by convention — this is expected and should be noted.

**Output fields:** `found`, `tools`, `config_files`, `lockfile_type` (e.g., `"npm"`, `"poetry"`, `"cargo"`), `evidence`

---

## Signal: `vcs_cli_tools`

**Purpose:** Version control CLI enables agents to interact with the repository platform.

**Detection:**

1. Check if `gh` CLI is available: run `which gh`
2. Check if git is available: run `which git`
3. For local scans, check if `gh auth status` succeeds (authenticated)

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `single_command_setup`

**Purpose:** One-command setup reduces onboarding friction.

**Detection:**

1. Search documentation for setup commands:
   - `README.md`, `CONTRIBUTING.md`, `CLAUDE.md` — look for sections like "Getting Started", "Setup", "Installation", "Quick Start"
   - Makefile targets: `setup`, `install`, `init`, `bootstrap`
   - Scripts: `scripts/setup.sh`, `scripts/bootstrap.sh`, `scripts/dev-setup.sh`, `bin/setup`
   - `./dev doctor` or `./dev setup` patterns
2. Check for `docker-compose up` as single-command setup
3. Check `package.json` scripts for `setup`, `prepare`, `postinstall`

**Output fields:** `found`, `tools`, `config_files`, `setup_commands` (list of setup commands found), `evidence`

---

## Signal: `fast_ci_feedback`

**Purpose:** CI under 10 minutes enables rapid iteration for agents.

**Detection:**

1. Check CI workflow files for timing indicators:
   - `timeout-minutes` settings in GitHub Actions workflows
   - Number and complexity of jobs/steps
   - Presence of caching (`actions/cache`, `setup-node` with cache)
2. Check recent CI run durations via GitHub API (remote only):
   - `gh api repos/{owner}/{repo}/actions/runs?per_page=5` — check `run_started_at` vs `updated_at`
3. If no CI detected, set `found: null`

**Output fields:** `found` (null if no CI), `tools`, `config_files`, `estimated_duration_minutes` (null if unknown), `evidence`

---

## Signal: `deployment_frequency`

**Purpose:** Regular releases indicate healthy release engineering.

**Detection (remote only):**

1. Check GitHub releases: `gh api repos/{owner}/{repo}/releases?per_page=10`
   - Count releases in last 90 days
   - Determine frequency (daily, weekly, monthly, sporadic)
2. Check git tags: `gh api repos/{owner}/{repo}/tags?per_page=10`
3. For local scans, set `found: null` with explanation

**Output fields:** `found` (null for local scans), `tools`, `config_files`, `recent_release_count` (in last 90 days), `evidence`

---

## Signal: `release_automation`

**Purpose:** Automated releases reduce human error and enable agent-triggered deployments.

**Detection:**

1. Search CI workflows for release jobs:
   - `.github/workflows/release.yml`, `publish.yml`, `deploy.yml`, `cd.yml`
   - Steps containing: `goreleaser`, `semantic-release`, `changesets/action`, `pypi`, `npm publish`, `cargo publish`, `twine upload`
   - Triggers: `on: release`, `on: push: tags:`
2. Search for release configs:
   - `.goreleaser.yml`, `.goreleaser.yaml`
   - `.releaserc`, `.releaserc.json`, `.releaserc.yml`, `release.config.js`
   - `.changeset/config.json`
   - `pyproject.toml` with `[tool.semantic_release]`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `release_notes_automation`

**Purpose:** Auto-generated changelogs document what shipped.

**Detection:**

1. Search for changelog generators:
   - `.goreleaser.yml` with `changelog` section
   - `cliff.toml`, `git-cliff` in CI
   - `.changeset/config.json` (changesets)
   - `towncrier.toml`, `pyproject.toml` with `[tool.towncrier]`
   - `.github/release.yml` (GitHub auto-generated release notes config)
   - `release-changelog-builder` action in CI workflows
   - `conventional-changelog` in package.json
2. Check for `CHANGELOG.md` or `CHANGES.md` (existence indicates manual or automated changelog)

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `automated_pr_review`

**Purpose:** Automated review bots provide instant feedback on PRs.

**Detection:**

1. Search CI workflows for review bots:
   - Steps using: `danger/danger-js`, `reviewdog`, `pullfrog`, `codeant-ai`, `copilot-review`
   - Actions: `actions/github-script` with review logic
2. Search for bot configs:
   - `dangerfile.js`, `dangerfile.ts`
   - `.github/workflows/` — workflows triggered on `pull_request_review`
3. Check repo for recent bot comments (remote only):
   - `gh api repos/{owner}/{repo}/pulls?per_page=5&state=closed` — look for bot usernames in comments

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `agentic_development`

**Purpose:** Agent-specific tooling directories and co-authorship indicate agent adoption.

**Detection:**

1. Search for agent config directories:
   - `.claude/` directory
   - `.claude/skills/` directory
   - `.factory/` directory
   - `.factory/skills/` directory
   - `.cursor/` directory
   - `.aider/` or `.aider.conf.yml`
   - `.coderabbit.yaml`
2. Search for agent instruction files:
   - `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.github/copilot-instructions.md`
3. Check git log for agent co-authorship (remote only):
   - `git log --all --format='%b' | grep -i 'co-authored-by.*\(claude\|copilot\|cursor\|coderabbit\|factory\)' | head -5`

**Output fields:** `found`, `tools`, `config_files`, `agent_configs` (list of agent-related files/dirs found), `evidence`

---

## Signal: `feature_flag_infra`

**Purpose:** Feature flags enable safe, incremental rollouts.

**Detection:**

1. Search dependency manifests:
   - `package.json`: `launchdarkly-node-server-sdk`, `@unleash/proxy-client-react`, `@statsig/js-client`, `flagsmith`, `@happykit/flags`, `@growthbook/growthbook`
   - `pyproject.toml` / `requirements.txt`: `launchdarkly-server-sdk`, `unleash-client-python`, `flagsmith`, `growthbook`
   - `go.mod`: `github.com/launchdarkly/go-server-sdk`, `github.com/Unleash/unleash-client-go`
   - `Cargo.toml`: `unleash-api-client`
2. Search source code for feature flag patterns:
   - `grep -r "feature_flag\|featureFlag\|isEnabled\|isFeatureEnabled\|feature_enabled" --include="*.go" --include="*.py" --include="*.ts" --include="*.js" --include="*.rs" -l | head -5`
3. Search for custom flag configs:
   - `features.yml`, `features.json`, `feature_flags/`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `build_performance_tracking`

**Purpose:** Build time tracking identifies and prevents build regressions.

**Detection:**

1. Search for build caching:
   - Bazel with `--remote_cache` or `--disk_cache` in `.bazelrc`
   - `turbo.json` (Turborepo caching)
   - `nx.json` with `tasksRunnerOptions` cache
   - `sccache` in CI or Rust build configs
   - `.ccache/` or `ccache` in CI
2. Search CI for build timing:
   - Actions: `benchmark-action`, `github-action-benchmark`
   - Steps exporting build timing metrics
3. Search for build profiling:
   - `--profile` flags in build commands
   - Webpack `speed-measure-webpack-plugin` in package.json

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `unused_deps_detection`

**Purpose:** Unused dependencies add bloat and potential security surface.

**Detection:**

1. Search for tools:
   - `package.json`: `depcheck`, `knip` (also does unused deps)
   - `pyproject.toml`: `deptry`
   - Go: check CI/Makefile for `go mod tidy` with `-diff` or verification step
   - Rust: `cargo-udeps` in CI
   - Check CI for `depcheck`, `deptry`, `cargo-udeps` steps
2. Search Makefile for targets: `check-deps`, `tidy`, `verify-deps`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `monorepo_tooling`

**Purpose:** Monorepo tools manage multi-package builds and dependencies.

**Detection:**

1. Search for monorepo managers:
   - `lerna.json` (Lerna)
   - `nx.json`, `project.json` (Nx)
   - `turbo.json` (Turborepo)
   - `rush.json` (Rush)
   - `pnpm-workspace.yaml` (pnpm workspaces)
   - `WORKSPACE`, `WORKSPACE.bazel` (Bazel)
   - `BUILD`, `BUILD.bazel` files (Bazel)
   - `Cargo.toml` root with `[workspace]` (Cargo workspaces)
   - `package.json` with `"workspaces"` key (npm/yarn workspaces)
   - `pants.toml` (Pants build)
   - `buck2` configs
2. This signal only applies if `project_structure.monorepo_detected` is true

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `dead_feature_flag_detection`

**Purpose:** Stale feature flags add complexity without value.

**Detection:**

1. Search for flag lifecycle tooling:
   - LaunchDarkly flag cleanup features
   - Custom scripts scanning for unused flags
   - CI steps checking for stale flags
2. Search for documentation about flag cleanup processes

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `heavy_dependency_detection`

**Purpose:** Large dependencies inflate bundle size and build times.

**Detection:**

1. Search for bundle analysis tools:
   - `package.json`: `bundlewatch`, `size-limit`, `@size-limit/preset-small-lib`, `webpack-bundle-analyzer`, `source-map-explorer`, `bundlephobia`
   - `.size-limit.json`, `.size-limit.js`
   - `bundlewatch.config.json`
2. Search CI for bundle size checks:
   - Steps running `size-limit`, `bundlewatch`, `bundle-analyzer`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `progressive_rollout`

**Purpose:** Canary/staged deployments reduce blast radius of changes.

**Detection:**

1. Search for deployment configs:
   - Kubernetes canary configs, `argo-rollouts` CRDs
   - `deploy/canary/`, `k8s/canary/`
   - CI workflows with staged deployment jobs (e.g., `deploy-canary`, `deploy-staging`, `deploy-production`)
   - Feature flag percentage-based rollout configs
2. Search for deployment tools:
   - `argo-rollouts`, `flagger`, `spinnaker` references in configs

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `rollback_automation`

**Purpose:** Automated rollback prevents prolonged outages from bad deploys.

**Detection:**

1. Search for rollback mechanisms:
   - CI workflows with rollback jobs or steps
   - Deployment scripts with rollback logic
   - `argo-rollouts` with automated rollback
   - Kubernetes deployment strategy with rollback annotations
2. Search docs for rollback procedures:
   - `RUNBOOK.md`, `docs/runbook/`, `docs/ops/` — grep for "rollback"

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `version_drift_detection`

**Purpose:** Detects inconsistent dependency versions across monorepo packages.

**Detection:**

1. Search for version sync tools:
   - `package.json`: `syncpack`, `manypkg`
   - `.syncpackrc`, `.syncpackrc.json`
   - `manypkg.config.js`
2. Search CI for version consistency checks:
   - Steps running `syncpack list-mismatches`, `manypkg check`

**Output fields:** `found`, `tools`, `config_files`, `evidence`
