# Agent-Fit: Agent-Readiness Assessment Skill

A Claude Code skill that evaluates your codebase's agent-readiness across 9 technical pillars, 86+ binary criteria, and 5 maturity levels. Produces an HTML report with pass rate scoring and a prioritized remediation roadmap.

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

An HTML report scoring your project 0-100% across 9 pillars:

| Pillar | Criteria | What It Measures |
|--------|----------|------------------|
| Style & Validation | 13 | Formatting, linting, type checking, complexity |
| Build System | 19 | CI/CD, dependencies, releases, automation |
| Testing | 8 | Coverage, isolation, flaky detection, performance |
| Documentation | 10 | README, AGENTS.md, API docs, architecture, changelog |
| Dev Environment | 5 | Devcontainer, env templates, local services |
| Debugging & Observability | 11 | Logging, tracing, metrics, alerting, profiling |
| Security & Governance | 13 | Branch protection, secrets, scanning, CODEOWNERS, SBOM |
| Task Discovery | 4 | Issue templates, labeling, backlog health |
| Product & Analytics | 3 | Analytics, error pipelines, A/B testing |

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
