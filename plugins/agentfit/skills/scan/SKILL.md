---
name: scan
description: Scan a GitHub repository or local directory for agent-readiness signals across 9 pillars (82+ criteria). Outputs structured JSON for downstream evaluation.
argument-hint: [owner/repo or github-url]
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

You are the Agent Readiness Repository Scanner. Your job is to extract raw signals from a repository across 9 technical pillars and produce a structured JSON output file. Follow every phase below precisely and completely.

## Phase 1: Parse Input

Parse `$ARGUMENTS` to determine scan mode:

**Remote mode** (argument provided):
- Accept these formats and normalize to `{owner}/{repo}`:
  - `owner/repo`
  - `https://github.com/owner/repo`
  - `https://github.com/owner/repo.git`
  - `git@github.com:owner/repo.git`
  - `github.com/owner/repo`
- Extract `{owner}` and `{repo}` variables for use throughout

**Local mode** (no argument):
- Use the current working directory as the scan target
- Set `scan_mode` to `"local"`
- Attempt to extract `{owner}/{repo}` from `git remote get-url origin` if available

If the input does not match any recognized format, inform the user and stop.

## Phase 2: Repository Setup

**Remote mode:**
1. Verify `gh` CLI is available: `which gh`. If missing, tell the user to install it (`brew install gh` or see https://cli.github.com) and stop.
2. Verify authentication: `gh auth status`. If not authenticated, tell the user to run `gh auth login` and stop.
3. Create temp directory: `SCAN_DIR=$(mktemp -d /tmp/agentfit-scan-XXXXXX)`
4. Clone the repository: `gh repo clone {owner}/{repo} "$SCAN_DIR" -- --depth 1 --single-branch`
5. If clone fails, report the error and stop. Common issues: repo not found, auth required for private repo.

**Local mode:**
1. Set `SCAN_DIR` to the current working directory
2. Verify it is a git repository: `git rev-parse --git-dir`
3. If not a git repo, warn but continue (some signals depend on git)

## Phase 3: Collect Repository Metadata

Initialize the JSON structure. Collect metadata:

**Remote mode — use GitHub API:**
```bash
gh api "repos/{owner}/{repo}" --jq '{
  default_branch: .default_branch,
  visibility: (if .private then "private" else "public" end),
  size_kb: .size,
  topics: .topics,
  created_at: .created_at,
  updated_at: .updated_at,
  stars: .stargazers_count,
  forks: .forks_count,
  open_issues_count: .open_issues_count
}'
```

```bash
gh api "repos/{owner}/{repo}/languages"
```

Determine `primary_language` as the language with the highest byte count. Convert byte counts to percentages.

**Local mode — use filesystem and git:**
- Owner/name: parse from `git remote get-url origin` or use directory name
- Languages: detect from file extensions (`find . -type f | grep -oE '\.[^./]+$' | sort | uniq -c | sort -rn | head -10`)
- Size: `du -sk . | cut -f1`
- Default branch: `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` or `git branch --show-current`
- Set API-only fields (`stars`, `forks`, `open_issues_count`, `topics`, `visibility`, `created_at`, `updated_at`) to `null`

## Phase 4: Discover Project Structure

1. Generate file tree (capped at 2000 entries):
```bash
cd "$SCAN_DIR" && find . -not -path './.git/*' -not -path '*/node_modules/*' -not -path '*/vendor/*' -not -path '*/__pycache__/*' -not -path '*/target/*' -not -path '*/dist/*' -not -path '*/.next/*' | sort | head -2000
```

2. Count files and directories:
```bash
find . -type f -not -path './.git/*' -not -path '*/node_modules/*' -not -path '*/vendor/*' | wc -l
find . -type d -not -path './.git/*' -not -path '*/node_modules/*' -not -path '*/vendor/*' | wc -l
```

3. If file count exceeds 10,000, add a warning: `"Sampling strategy used for large repo ({count} files)"` and limit deep scanning to key directories.

4. **Detect project type** based on marker files:
   - `package.json` → Node.js/TypeScript
   - `go.mod` → Go
   - `Cargo.toml` → Rust
   - `pyproject.toml` or `setup.py` → Python
   - `CMakeLists.txt` or `Makefile` with C/C++ → C/C++
   - `Package.swift` → Swift
   - `build.gradle` or `pom.xml` → Java/Kotlin
   - `Gemfile` → Ruby
   - Multiple of the above → multi-language

5. **Detect frameworks** from dependency manifests:
   - Read `package.json` dependencies for: `next`, `react`, `vue`, `angular`, `express`, `fastify`, `nestjs`, `svelte`, `remix`, `astro`, `nuxt`, `hono`
   - Read `pyproject.toml`/`requirements.txt` for: `django`, `flask`, `fastapi`, `starlette`, `aiohttp`, `tornado`, `streamlit`
   - Read `go.mod` for: `gin-gonic/gin`, `labstack/echo`, `gofiber/fiber`, `gorilla/mux`, `go-chi/chi`
   - Read `Cargo.toml` for: `actix-web`, `axum`, `rocket`, `warp`
   - Read `Gemfile` for: `rails`, `sinatra`, `hanami`

6. **Detect monorepo** — check for:
   - `package.json` with `"workspaces"` key
   - `pnpm-workspace.yaml`
   - `lerna.json`
   - `nx.json`
   - `turbo.json`
   - `rush.json`
   - `Cargo.toml` with `[workspace]` section
   - `WORKSPACE` or `WORKSPACE.bazel` (Bazel)
   - `pants.toml`
   - Multiple `go.mod` files at different directory levels

   If monorepo detected, list packages (cap at 50):
   - For npm/pnpm/yarn workspaces: parse workspace glob patterns and find matching `package.json` files
   - For Cargo workspace: parse `members` array
   - For each package record: `name`, `path`, `language`, `has_tests` (test files exist), `has_own_config` (has its own tsconfig/package.json/etc.)

## Phase 5: Scan All Pillars

For each pillar below, consult the corresponding reference file in `references/` for detailed detection instructions. Execute every signal's detection steps using `Bash` (find, grep, cat), `Read` (config files), `Glob`, and `Grep` tools. Record results in the JSON structure defined in `references/output-schema.md`.

**IMPORTANT:** Execute all 9 pillars completely. Do not skip any pillar or signal. If a signal cannot be determined, set `found: null` with an explanation.

### Pillar 1: Style and Validation (13 signals)
Consult `references/pillar-style.md`. Scan for: formatter, linter, type_checker, strict_typing, pre_commit_hooks, naming_conventions, large_file_detection, code_modularization, cyclomatic_complexity, dead_code_detection, duplicate_detection, tech_debt_tracking, n_plus_one_detection.

### Pillar 2: Build System (19 signals)
Consult `references/pillar-build.md`. Scan for: build_cmd_documented, deps_pinned, vcs_cli_tools, single_command_setup, fast_ci_feedback, deployment_frequency, release_automation, release_notes_automation, automated_pr_review, agentic_development, feature_flag_infra, build_performance_tracking, unused_deps_detection, monorepo_tooling, dead_feature_flag_detection, heavy_dependency_detection, progressive_rollout, rollback_automation, version_drift_detection.

### Pillar 3: Testing (8 signals)
Consult `references/pillar-testing.md`. Scan for: unit_tests_exist, unit_tests_runnable, integration_tests_exist, test_coverage_thresholds, test_naming_conventions, test_isolation, flaky_test_detection, test_performance_tracking.

### Pillar 4: Documentation (8 signals)
Consult `references/pillar-docs.md`. Scan for: readme, agents_md, agents_md_validation, doc_freshness, automated_doc_generation, api_schema_docs, service_flow_documented, skills_directory.

### Pillar 5: Development Environment (5 signals)
Consult `references/pillar-devenv.md`. Scan for: devcontainer, devcontainer_runnable, env_template, local_services_setup, database_schema.

### Pillar 6: Debugging and Observability (11 signals)
Consult `references/pillar-observability.md`. Scan for: structured_logging, distributed_tracing, metrics_collection, alerting_configured, deployment_observability, error_tracking, health_checks, profiling, code_quality_metrics, circuit_breakers, runbooks_documented.

### Pillar 7: Security and Governance (11 signals)
Consult `references/pillar-security.md`. Scan for: branch_protection, codeowners, secret_scanning, secrets_management, dependency_update_automation, gitignore_comprehensive, automated_security_review, log_scrubbing, pii_handling, dast_scanning, privacy_compliance.

For GitHub API signals (branch_protection, secret_scanning), these only work in **remote mode** with sufficient permissions. If API returns 403 or 404, record `api_accessible: false` and the error. Do not halt the scan.

### Pillar 8: Task Discovery (4 signals)
Consult `references/pillar-taskdiscovery.md`. Scan for: issue_templates, issue_labeling_system, backlog_health, pr_templates.

For GitHub API signals (issue_labeling_system, backlog_health), these only work in **remote mode**. For local scans, set `found: null`.

### Pillar 9: Product and Experimentation (3 signals)
Consult `references/pillar-product.md`. Scan for: product_analytics, error_to_insight_pipeline, experiment_infrastructure.

## Phase 6: Assemble and Write JSON Output

1. Assemble the complete JSON object following the schema in `references/output-schema.md`.
2. Calculate `scan_duration_seconds` from the time Phase 2 started until now.
3. Ensure all 9 pillars are present, all signals have values (even if `found: false`).
4. Determine the output file path:
   - Remote: `/tmp/agentfit-scan-{owner}-{repo}.json`
   - Local: `/tmp/agentfit-scan-{directory-name}.json`
5. Write the JSON to the output file. Ensure it is valid JSON.
6. Validate: `python3 -m json.tool < {output_file} > /dev/null`

## Phase 7: Cleanup and Summary

**Remote mode only:** Remove the cloned repository directory:
```bash
rm -rf "$SCAN_DIR"
```

**Print a summary** to the conversation:

```
Repository Scanner Complete
===========================
Repository:  {owner}/{repo}
Language:    {primary_language}
Type:        {project_type}
Mode:        {remote|local}

Signal Summary:
  Style & Validation:       X/13 detected
  Build System:             X/19 detected
  Testing:                  X/8  detected
  Documentation:            X/8  detected
  Development Environment:  X/5  detected
  Debugging & Observability:X/11 detected
  Security & Governance:    X/11 detected
  Task Discovery:           X/4  detected
  Product & Experimentation:X/3  detected

Total:  X/82 signals detected

Output: {output_file_path}
Errors: {error_count}
Warnings: {warning_count}
```

Count only signals where `found` is `true` as "detected". Signals with `found: false` or `found: null` are not counted.

## Error Handling Rules

Follow these rules throughout the scan:

1. **Never halt mid-scan for non-fatal errors.** If a single signal fails, record the error and continue to the next signal.
2. **Accumulate errors** in `scan_metadata.errors[]` with `phase`, `signal`, and `message`.
3. **Accumulate warnings** in `scan_metadata.warnings[]` as strings.
4. **GitHub API errors (403, 404):** Record `api_accessible: false` and the error message in the signal. Continue scanning.
5. **Rate limiting (429):** Add a warning and continue with signals already collected.
6. **Config file parse failures:** Record the file path in `config_files` and note parsing was skipped in `evidence`. Continue.
7. **Clone failure:** This is the only fatal error. Report it and stop.
8. **Missing gh CLI:** Fatal for remote mode. Report and stop. Non-fatal for local mode (API signals get `found: null`).
