# Scanner Output JSON Schema

This document defines the complete output schema for the repository scanner. All scanner output MUST conform to this structure.

## Top-Level Structure

```json
{
  "schema_version": "1.0.0",
  "scan_metadata": { },
  "repository": { },
  "project_structure": { },
  "pillars": { }
}
```

---

## `scan_metadata`

```json
{
  "timestamp": "2026-05-08T12:00:00Z",
  "scanner_version": "0.2.0",
  "scan_duration_seconds": 45,
  "scan_mode": "remote | local",
  "repo_cloned_to": "/tmp/agentfit-scan-XXXXXX | null",
  "errors": [
    { "phase": "github_api", "signal": "branch_protection", "message": "403 Forbidden" }
  ],
  "warnings": [
    "File tree truncated at 2000 entries",
    "Sampling strategy used for large repo (15432 files)"
  ]
}
```

---

## `repository`

For **remote** scans, populate from `gh api repos/{owner}/{repo}` and `gh api repos/{owner}/{repo}/languages`. For **local** scans, derive from git remote URL and filesystem stats.

```json
{
  "owner": "expressjs",
  "name": "express",
  "full_name": "expressjs/express",
  "url": "https://github.com/expressjs/express",
  "default_branch": "main",
  "visibility": "public | private",
  "size_kb": 12345,
  "topics": ["nodejs", "web-framework"],
  "languages": {
    "JavaScript": 85.2,
    "TypeScript": 14.8
  },
  "primary_language": "JavaScript",
  "created_at": "2010-05-27T00:00:00Z",
  "updated_at": "2026-05-01T00:00:00Z",
  "stars": 65000,
  "forks": 16000,
  "open_issues_count": 180
}
```

For local scans where GitHub API is unavailable, set API-derived fields to `null`.

---

## `project_structure`

```json
{
  "type": "monorepo | single-app | library | cli | web-app | api | framework | unknown",
  "monorepo_detected": false,
  "packages": [
    {
      "name": "web",
      "path": "apps/web",
      "language": "TypeScript",
      "has_tests": true,
      "has_own_config": true
    }
  ],
  "frameworks_detected": ["express", "react", "django"],
  "total_files": 456,
  "total_directories": 78,
  "key_directories": ["src/", "test/", "docs/", ".github/"],
  "key_files": ["package.json", "tsconfig.json", "README.md"],
  "file_tree_sample": [
    "package.json",
    "tsconfig.json",
    "src/",
    "src/index.ts",
    "test/",
    "test/app.test.ts"
  ]
}
```

`packages` is only populated when `monorepo_detected` is `true`. Cap at 50 entries.

`file_tree_sample` is capped at 500 entries (the most structurally informative paths).

---

## `pillars`

Contains 9 pillar objects. Each pillar has a `signals` map keyed by signal name. Every signal follows a base structure with criterion-specific extensions.

### Base Signal Structure

```json
{
  "found": true,
  "tools": ["prettier", "eslint"],
  "config_files": [".prettierrc", "eslint.config.js"],
  "evidence": "Short description of what was found"
}
```

- `found`: `true` if the signal is detected, `false` if not, `null` if detection was skipped or errored
- `tools`: List of specific tool names detected (empty array if none)
- `config_files`: List of config file paths found relative to repo root (empty array if none)
- `evidence`: Brief human-readable summary of what was found or why it was not found

---

### Pillar 1: `style_and_validation`

```json
{
  "signals": {
    "formatter":              { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "linter":                 { "found": bool, "tools": [], "config_files": [], "rules_configured": bool, "evidence": "" },
    "type_checker":           { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "strict_typing":          { "found": bool, "tools": [], "config_files": [], "strict_settings": {}, "evidence": "" },
    "pre_commit_hooks":       { "found": bool, "tools": [], "config_files": [], "hooks_list": [], "evidence": "" },
    "naming_conventions":     { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "large_file_detection":   { "found": bool, "tools": [], "config_files": [], "lfs_configured": bool, "evidence": "" },
    "code_modularization":    { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "cyclomatic_complexity":  { "found": bool, "tools": [], "config_files": [], "thresholds": {}, "evidence": "" },
    "dead_code_detection":    { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "duplicate_detection":    { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "tech_debt_tracking":     { "found": bool, "tools": [], "config_files": [], "ci_scanning": bool, "evidence": "" },
    "n_plus_one_detection":   { "found": null, "tools": [], "config_files": [], "skipped_reason": "No database layer detected", "evidence": "" }
  }
}
```

---

### Pillar 2: `build_system`

```json
{
  "signals": {
    "build_cmd_documented":       { "found": bool, "tools": [], "config_files": [], "documented_commands": [], "evidence": "" },
    "deps_pinned":                { "found": bool, "tools": [], "config_files": [], "lockfile_type": "", "evidence": "" },
    "vcs_cli_tools":              { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "single_command_setup":       { "found": bool, "tools": [], "config_files": [], "setup_commands": [], "evidence": "" },
    "fast_ci_feedback":           { "found": null, "tools": [], "config_files": [], "estimated_duration_minutes": null, "evidence": "" },
    "deployment_frequency":       { "found": null, "tools": [], "config_files": [], "recent_release_count": null, "evidence": "" },
    "release_automation":         { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "release_notes_automation":   { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "automated_pr_review":        { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "agentic_development":        { "found": bool, "tools": [], "config_files": [], "agent_configs": [], "evidence": "" },
    "feature_flag_infra":         { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "build_performance_tracking": { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "unused_deps_detection":      { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "monorepo_tooling":           { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "dead_feature_flag_detection":{ "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "heavy_dependency_detection": { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "progressive_rollout":        { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "rollback_automation":        { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "version_drift_detection":    { "found": bool, "tools": [], "config_files": [], "evidence": "" }
  }
}
```

---

### Pillar 3: `testing`

```json
{
  "signals": {
    "unit_tests_exist":         { "found": bool, "tools": [], "config_files": [], "test_file_count": 0, "sample_paths": [], "evidence": "" },
    "unit_tests_runnable":      { "found": bool, "tools": [], "config_files": [], "test_command": "", "evidence": "" },
    "integration_tests_exist":  { "found": bool, "tools": [], "config_files": [], "test_file_count": 0, "sample_paths": [], "evidence": "" },
    "test_coverage_thresholds": { "found": bool, "tools": [], "config_files": [], "thresholds": {}, "evidence": "" },
    "test_naming_conventions":  { "found": bool, "tools": [], "config_files": [], "patterns": [], "evidence": "" },
    "test_isolation":           { "found": bool, "tools": [], "config_files": [], "parallel_execution": bool, "evidence": "" },
    "flaky_test_detection":     { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "test_performance_tracking":{ "found": bool, "tools": [], "config_files": [], "evidence": "" }
  }
}
```

---

### Pillar 4: `documentation`

```json
{
  "signals": {
    "readme":                  { "found": bool, "tools": [], "config_files": [], "size_bytes": 0, "has_install": bool, "has_usage": bool, "has_contributing": bool, "evidence": "" },
    "agents_md":               { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "agents_md_validation":    { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "doc_freshness":           { "found": bool, "tools": [], "config_files": [], "last_modified_days": 0, "evidence": "" },
    "automated_doc_generation":{ "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "api_schema_docs":         { "found": bool, "tools": [], "config_files": [], "schema_types": [], "evidence": "" },
    "service_flow_documented": { "found": bool, "tools": [], "config_files": [], "diagram_types": [], "evidence": "" },
    "skills_directory":        { "found": bool, "tools": [], "config_files": [], "skill_count": 0, "evidence": "" }
  }
}
```

---

### Pillar 5: `development_environment`

```json
{
  "signals": {
    "devcontainer":         { "found": bool, "tools": [], "config_files": [], "base_image": "", "evidence": "" },
    "devcontainer_runnable":{ "found": null, "tools": [], "config_files": [], "skipped_reason": "Requires container runtime", "evidence": "" },
    "env_template":         { "found": bool, "tools": [], "config_files": [], "variable_count": 0, "evidence": "" },
    "local_services_setup": { "found": bool, "tools": [], "config_files": [], "services": [], "evidence": "" },
    "database_schema":      { "found": bool, "tools": [], "config_files": [], "migration_tool": "", "migration_count": 0, "evidence": "" }
  }
}
```

---

### Pillar 6: `debugging_and_observability`

```json
{
  "signals": {
    "structured_logging":        { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "distributed_tracing":       { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "metrics_collection":        { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "alerting_configured":       { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "deployment_observability":  { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "error_tracking":            { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "health_checks":             { "found": bool, "tools": [], "config_files": [], "endpoints": [], "evidence": "" },
    "profiling":                 { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "code_quality_metrics":      { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "circuit_breakers":          { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "runbooks_documented":       { "found": bool, "tools": [], "config_files": [], "evidence": "" }
  }
}
```

---

### Pillar 7: `security_and_governance`

```json
{
  "signals": {
    "branch_protection":          { "found": null, "tools": [], "config_files": [], "api_accessible": bool, "rules": {}, "error": null, "evidence": "" },
    "codeowners":                 { "found": bool, "tools": [], "config_files": [], "team_count": 0, "evidence": "" },
    "secret_scanning":            { "found": null, "tools": [], "config_files": [], "api_accessible": bool, "enabled": null, "error": null, "evidence": "" },
    "secrets_management":         { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "dependency_update_automation": { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "gitignore_comprehensive":    { "found": bool, "tools": [], "config_files": [], "covers_env": bool, "covers_credentials": bool, "covers_build": bool, "evidence": "" },
    "automated_security_review":  { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "log_scrubbing":              { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "pii_handling":               { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "dast_scanning":              { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "privacy_compliance":         { "found": bool, "tools": [], "config_files": [], "evidence": "" }
  }
}
```

---

### Pillar 8: `task_discovery`

```json
{
  "signals": {
    "issue_templates":       { "found": bool, "tools": [], "config_files": [], "template_types": [], "evidence": "" },
    "issue_labeling_system": { "found": null, "tools": [], "config_files": [], "api_accessible": bool, "label_count": 0, "labels": [], "error": null, "evidence": "" },
    "backlog_health":        { "found": null, "tools": [], "config_files": [], "api_accessible": bool, "sample_size": 0, "titled_pct": 0, "labeled_pct": 0, "avg_body_length": 0, "error": null, "evidence": "" },
    "pr_templates":          { "found": bool, "tools": [], "config_files": [], "evidence": "" }
  }
}
```

---

### Pillar 9: `product_and_experimentation`

```json
{
  "signals": {
    "product_analytics":        { "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "error_to_insight_pipeline":{ "found": bool, "tools": [], "config_files": [], "evidence": "" },
    "experiment_infrastructure":{ "found": bool, "tools": [], "config_files": [], "evidence": "" }
  }
}
```

---

## Notes

- All file paths in `config_files` are relative to repository root.
- `found: null` means detection was skipped (not applicable or errored). Always include `skipped_reason` or `error` explaining why.
- `tools` lists specific tool names (e.g., `"prettier"`, `"eslint"`, `"golangci-lint"`), not categories.
- `evidence` is a brief human-readable sentence. Keep under 200 characters.
- The entire JSON must be valid and parseable by `python3 -m json.tool`.
