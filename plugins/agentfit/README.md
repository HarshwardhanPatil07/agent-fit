# Agent-Fit: Agent-Readiness Assessment

A Claude Code plugin that analyzes repositories for agent-readiness. Two commands: a quick local assessment and a comprehensive 82-signal scanner.

## Install

Add the following to your Claude Code settings (`.claude/settings.json` or global settings):

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

## Commands

### `/agentfit` -- Quick Local Assessment

Run inside any project directory for a fast 5-dimension readiness report:

```
/agentfit
```

Produces a 0-10 score across machine-readable interfaces, authentication model, documentation, discoverability, and agent experience. Each finding is tagged `FOUND`, `PARTIAL`, or `MISSING` with prioritized recommendations.

### `/agentfit:scan` -- Comprehensive Repository Scanner

Scan any GitHub repository (or your local directory) for 82+ agent-readiness signals across 9 technical pillars:

```
/agentfit:scan owner/repo
/agentfit:scan https://github.com/owner/repo
/agentfit:scan
```

**Accepted input formats:**
- `owner/repo` -- GitHub shorthand
- `https://github.com/owner/repo` -- HTTPS URL
- `git@github.com:owner/repo.git` -- SSH URL
- No argument -- scans the current working directory

**What it produces:**

A structured JSON file at `/tmp/agentfit-scan-{repo}.json` containing raw signals across:

| Pillar | Signals | What It Covers |
|--------|---------|----------------|
| Style & Validation | 13 | Formatting, linting, type checking, complexity analysis |
| Build System | 19 | CI/CD, dependency management, releases, caching |
| Testing | 8 | Coverage, isolation, flaky detection, performance |
| Documentation | 8 | README, AGENTS.md, API docs, architecture diagrams |
| Dev Environment | 5 | Devcontainer, env templates, local services, schemas |
| Debugging & Observability | 11 | Logging, tracing, metrics, alerting, profiling |
| Security & Governance | 11 | Branch protection, secrets, scanning, CODEOWNERS |
| Task Discovery | 4 | Issue templates, labeling, backlog health, PR templates |
| Product & Experimentation | 3 | Analytics, error pipelines, A/B testing |

The scanner also prints a summary to the conversation showing signal counts per pillar.

**Features:**
- Supports public and private GitHub repositories (via `gh` CLI auth)
- Detects monorepos and lists packages
- Identifies languages, frameworks, and project type
- Handles GitHub API settings (branch protection, secret scanning, labels)
- Graceful error handling -- partial results when API access is limited

**Requirements for remote scans:**
- `gh` CLI installed and authenticated (`gh auth login`)

## Requirements

- Claude Code (no additional dependencies for local assessment)
- `gh` CLI for remote repository scanning

## Team

Built for Agent Skill Wars. See `Agents-First-Software-Vision.md` at repo root for full design rationale.
