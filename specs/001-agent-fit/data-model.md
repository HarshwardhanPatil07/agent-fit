# Data Model: Agent-Fit Assessment Skill

**Date**: 2026-05-11
**Feature**: 001-agent-fit

## Entities

### AssessmentReport

The top-level output entity.

| Field | Type | Description |
|-------|------|-------------|
| project_name | string | Repository/project name (from directory name or git remote) |
| language | string | Primary language detected (Go, Python, TypeScript, Rust, C++, Swift) |
| repo_path | string | GitHub org/repo path (from git remote) or local directory path |
| pass_rate | number | Overall pass rate 0-100% (criteria_passed / applicable_criteria * 100) |
| maturity_level | number | 1-5, determined by gated 80% progression |
| maturity_label | string | Functional, Documented, Standardized, Optimized, or Autonomous |
| summary_headline | string | Narrative headline (e.g., "Strong Testing") |
| summary_text | string | 1-2 sentence narrative summary |
| pillars | Pillar[] | Array of 9 pillar results |
| strengths | Highlight[] | Top 3 strengths (highest-scoring areas) |
| opportunities | Highlight[] | Top 3 opportunities (highest-impact gaps) |
| level_progress | LevelProgress[] | L1-L5 completion percentages |

### Pillar

| Field | Type | Description |
|-------|------|-------------|
| name | string | Display name (e.g., "Style & Validation") |
| criteria_passed | number | Count of criteria with FOUND status |
| criteria_total | number | Count of applicable criteria (excludes skipped) |
| percentage | number | (criteria_passed / criteria_total) * 100 |
| criteria | Criterion[] | Array of criterion results |

### Criterion

| Field | Type | Description |
|-------|------|-------------|
| name | string | snake_case identifier (e.g., `pre_commit_hooks`) |
| status | enum | `found` (✓), `missing` (✗), or `skipped` (—) |
| score | string | "1/1", "0/1", or "—/—" |
| evidence | string | Human-readable explanation with file paths or tool names |
| level | number | Which maturity level this criterion belongs to (1-5) |
| signal_type | enum | How this criterion is evaluated |

### Signal Types (enum)

| Value | Description | Example |
|-------|-------------|---------|
| file_existence | Check if a specific file/directory exists | `.pre-commit-config.yaml` |
| config_parsing | Read a config file and check for specific settings | `tsconfig.json` → `strict: true` |
| ci_workflow | Read CI workflow files for specific steps | `.github/workflows/*.yml` |
| dependency_check | Check dependency manifests for specific packages | `go.mod` → `go.uber.org/zap` |
| code_search | Grep source code for patterns | `grep -r "health" --include="*.go"` |
| api_check | Use `gh` CLI to query GitHub API | `gh api repos/{owner}/{repo}/rules` |
| git_history | Query git log for patterns | `git log --format="%aN" -100` |
| doc_content | Read documentation and check for specific content | README.md contains build commands |

### Highlight

| Field | Type | Description |
|-------|------|-------------|
| number | number | Display order (01, 02, 03) |
| title | string | Short title (e.g., "Testing (100%)") |
| detail | string | Supporting evidence sentence |

### LevelProgress

| Field | Type | Description |
|-------|------|-------------|
| level | number | 1-5 |
| label | string | L1, L2, L3, L4, L5 |
| percentage | number | Completion percentage for this level's criteria |

## Criterion-to-Level Mapping

### Level 1: Functional (baseline)
- formatter, lint_config, type_check, strict_typing
- unit_tests_exist, unit_tests_runnable, test_naming_conventions
- readme, documentation_freshness
- build_cmd_doc, deps_pinned, vcs_cli_tools
- gitignore_comprehensive

### Level 2: Documented
- pre_commit_hooks, naming_consistency
- agents_md, skills
- devcontainer, env_template, local_services_setup, database_schema
- single_command_setup
- branch_protection, codeowners, secrets_management
- issue_templates, pr_templates
- structured_logging

### Level 3: Standardized
- large_file_detection, code_modularization, cyclomatic_complexity
- dead_code_detection, duplicate_code_detection, tech_debt_tracking
- integration_tests_exist, test_coverage_thresholds, test_isolation
- agents_md_validation, automated_doc_generation, api_schema_docs
- service_flow_documented
- distributed_tracing, metrics_collection, health_checks
- code_quality_metrics
- secret_scanning, dependency_update_automation
- automated_security_review, log_scrubbing
- issue_labeling_system, backlog_health
- fast_ci_feedback, release_automation, release_notes_automation
- unused_deps_detection

### Level 4: Optimized
- n_plus_one_detection
- deployment_frequency, automated_pr_review, agentic_development
- feature_flag_infrastructure, build_performance_tracking
- monorepo_tooling
- flaky_test_detection, test_performance_tracking
- devcontainer_runnable
- alerting_configured, deployment_observability
- error_tracking_contextualized, profiling_instrumentation
- circuit_breakers, runbooks_documented
- pii_handling

### Level 5: Autonomous
- dead_feature_flag_detection, heavy_dependency_detection
- progressive_rollout, rollback_automation, version_drift_detection
- product_analytics_instrumentation
- error_to_insight_pipeline, experiment_infrastructure
- dast_scanning, privacy_compliance

## Relationships

```
AssessmentReport 1───* Pillar 1───* Criterion
AssessmentReport 1───* Highlight
AssessmentReport 1───* LevelProgress
```

## State Transitions

No state transitions — the assessment is a single-run, stateless evaluation.
Each run produces a complete report from scratch.
