# Agent-Fit: Agent-Readiness Assessment Skill

A Claude Code skill that evaluates your codebase's agent-readiness across 9 technical pillars, 75+ binary criteria, and 5 maturity levels. Produces an HTML report with pass rate scoring, weighted scoring, and project-type-aware skip rules.

## Install

Add to your Claude Code settings (`.claude/settings.json` or global):

```json
{
  "extraKnownMarketplaces": {
    "agent-fit": {
      "source": {
        "source": "git",
        "url": "git@github.com:HarshwardhanPatil07/agent-fit.git"
      }
    }
  },
  "enabledPlugins": {
    "agentfit@agent-fit": true
  }
}
```

## Run

Inside Claude Code, in any project directory:

```
/agentfit
```

To fix missing criteria automatically:

```
/agentfit-fix
/agentfit-fix --top 3
```

## What You Get

An HTML report scoring your project 0-100% across 9 pillars:

| Pillar | Criteria | What It Measures |
|--------|----------|------------------|
| Style & Validation | 13 | Formatting, linting, type checking, complexity |
| Build System | 19 | CI/CD, dependencies, releases, automation |
| Testing | 8 | Coverage, isolation, flaky detection, performance |
| Documentation | 8 | README, AGENTS.md, API docs, architecture |
| Dev Environment | 5 | Devcontainer, env templates, local services |
| Debugging & Observability | 11 | Logging, tracing, metrics, alerting, profiling |
| Security & Governance | 11 | Branch protection, secrets, scanning, CODEOWNERS |
| Task Discovery | 4 | Issue templates, labeling, backlog health |
| Product & Analytics | 3 | Analytics, error pipelines, A/B testing |

## v2 Features

### Project-Type Detection

The skill automatically detects your project type (library, CLI, web app, API service, monorepo) and skips irrelevant criteria. A CLI project won't be penalized for missing DAST scanning or health checks.

Multiple types can be detected simultaneously (e.g., a monorepo containing a CLI and a web app). Criteria are only skipped if ALL detected types would skip them.

### Custom Criteria via `.agentfit.yml`

Add project-specific criteria or disable irrelevant ones by creating `.agentfit.yml` at your repo root:

```yaml
custom_criteria:
  - name: "internal_api_docs"
    pillar: "Documentation"
    level: "L2"
    check: "docs/api/ directory exists"
    found_when: "docs/api/ contains at least one file"

disabled_criteria:
  - "product_analytics_instrumentation"
```

See `examples/agentfit.yml` for a full example.

### Weighted Scoring

Alongside the unweighted pass rate, the report shows a weighted score based on criteria impact tiers (high=3x, medium=2x, low=1x). High-impact criteria like type checking and unit tests count more than low-impact ones like duplicate code detection.

### Skill Versioning

The HTML report footer and JSON output include the skill version (`schema_version`), assessment date, and git SHA. JSON consumers can detect schema changes across versions.

### Automated Remediation (`/agentfit-fix`)

After running `/agentfit`, run `/agentfit-fix` to generate files for missing criteria (AGENTS.md, .pre-commit-config.yaml, devcontainer, dependabot.yml, etc.). Use `--top N` to limit to the N highest-priority fixes.

## CI Integration (GitHub Actions)

Copy `examples/github-action.yml` to `.github/workflows/agentfit.yml` in your repo. The workflow:
- Runs `/agentfit` on every pull request
- Uploads JSON artifact for trend tracking
- Posts a summary comment on the PR with pass rate and maturity level

## Maturity Levels

| Level | Name | What It Means |
|-------|------|---------------|
| L1 | Functional | Code runs but requires manual setup |
| L2 | Documented | Basic docs and processes exist |
| L3 | Standardized | Processes enforced through automation |
| L4 | Optimized | Fast feedback loops, data-driven improvement |
| L5 | Autonomous | Self-improving systems |

To unlock a level, pass 80% of that level's criteria AND all previous levels.

## Supported Languages

Go, Python, TypeScript/JavaScript, Rust, C++, Swift

## Optional: GitHub API Criteria

Some criteria (branch protection, issue labels, secret scanning) use the `gh` CLI. If `gh` is not installed or authenticated, those criteria are gracefully skipped.

## Requirements

- Claude Code (that's it — no dependencies, no API keys, no installs)
