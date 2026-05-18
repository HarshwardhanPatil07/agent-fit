# Agent-Fit: Agent-Readiness Assessment Skill

A Claude Code skill that evaluates your codebase's agent-readiness across 9 technical pillars, 80+ binary criteria, and 5 maturity levels. Produces an HTML report plus a JSON sidecar for automation.

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

## What You Get

An HTML report and matching JSON sidecar scoring your project 0-100% across 9 pillars:

| Pillar | Criteria | What It Measures |
|--------|----------|------------------|
| Style & Validation | 13 | Formatting, linting, type checking, complexity |
| Build System | 19 | CI/CD, dependencies, releases, automation |
| Testing | 8 | Coverage, isolation, flaky detection, performance |
| Documentation | 10 | README, AGENTS.md, API docs, architecture |
| Dev Environment | 5 | Devcontainer, env templates, local services |
| Debugging & Observability | 11 | Logging, tracing, metrics, alerting, profiling |
| Security & Governance | 13 | Branch protection, secrets, scanning, CODEOWNERS |
| Task Discovery | 4 | Issue templates, labeling, backlog health |
| Product & Analytics | 3 | Analytics, error pipelines, A/B testing |

### Output Files

- HTML: `/tmp/agentfit-report-{project_name}.html`
- JSON: `/tmp/agentfit-report-{project_name}.json`

The JSON sidecar includes report metadata, pillar and criterion details, level progress, gate status, and delta fields for trend automation.

### Report UX Features

- Collapsible pillar sections
- Status filters: `Show All`, `Found`, `Missing`, `Skipped`
- Criterion name search
- Level filter (L1-L5)
- Metadata footer (timestamp, version, criteria totals, git SHA, duration)
- Delta highlighting for criteria that changed since the previous JSON baseline

## Maturity Levels

| Level | Name | What It Means |
|-------|------|---------------|
| L1 | Functional | Code runs but requires manual setup |
| L2 | Documented | Basic docs and processes exist |
| L3 | Standardized | Processes enforced through automation |
| L4 | Optimized | Fast feedback loops, data-driven improvement |
| L5 | Autonomous | Self-improving systems |

To unlock a level, pass 80% of that level's criteria AND all previous levels.

Console output also includes per-level gate status (`✓`, `✗`, `— blocked`) so CI logs show why a level is locked.

## Supported Languages

Go, Python, TypeScript/JavaScript, Rust, C++, Swift

## Optional: GitHub API Criteria

Some criteria (branch protection, issue labels, secret scanning) use the `gh` CLI. If `gh` is not installed or authenticated, those criteria are gracefully skipped.

## Trend Comparison

When `/tmp/agentfit-report-{project_name}.json` exists from a previous run, Agent-Fit computes pass-rate delta and highlights criterion status changes. If no baseline is available (or baseline is malformed), the run still succeeds and reports current-state results.

## Requirements

- Claude Code (that's it — no dependencies, no API keys, no installs)
