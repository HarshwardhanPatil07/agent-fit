# Quickstart: Agent-Fit v2 Features

## Project-Type Detection

No configuration needed. Run `/agentfit` and the skill automatically detects your project type (library, CLI, web app, API service, monorepo) and skips irrelevant criteria.

The detected project type appears in the report header alongside the language badge.

## Custom Criteria via `.agentfit.yml`

1. Create `.agentfit.yml` at your repo root:

```yaml
custom_criteria:
  - name: "my_custom_check"
    pillar: "Documentation"
    level: "L2"
    check: "Custom documentation requirement"
    found_when: "docs/custom/ directory contains files"

disabled_criteria:
  - "product_analytics_instrumentation"
```

2. Run `/agentfit` — your custom criterion appears in the report and the disabled one is removed.

## Weighted Scoring

Automatically displayed in the report header alongside the unweighted pass rate. No configuration needed.

## Skill Version & Schema

The report footer now shows the skill version, assessment date, and git SHA. JSON output includes `schema_version` for tooling consumers.

## Automated Remediation

1. Run `/agentfit` first (generates JSON sidecar).
2. Run `/agentfit-fix` to generate files for missing criteria.
3. Choose: commit to current branch, create new branch, or leave unstaged.
4. Use `--top N` to limit to the N highest-priority fixes.

## CI Integration (GitHub Actions)

Copy `plugins/agentfit/examples/github-action.yml` to `.github/workflows/agentfit.yml` in your repo. The workflow runs on every PR and posts a summary comment.
