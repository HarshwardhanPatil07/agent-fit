# Agent-Fit

A Claude Code skill that scans your codebase and tells you how agent-fit your software is.

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

## Usage

Inside Claude Code:

```
/agentfit                              # scan current directory
/agentfit expressjs/express            # scan a remote GitHub repo
/agentfit https://github.com/owner/repo  # scan via full URL
```

Remote scanning requires the `gh` CLI installed and authenticated (`gh auth login`).

## What You Get

A structured agent-fit report scoring your project 0-10 across 5 dimensions:

- **Machine-Readable Interfaces** (0-3) -- APIs, OpenAPI specs, MCP, CLI, webhooks, SDKs
- **Authentication Model** (0-3) -- M2M auth, API keys, credential management
- **Documentation for Agents** (0-2) -- AGENTS.md, OpenAPI completeness, error/rate-limit docs
- **Discoverability** (0-1) -- MCP card, API catalog, robots.txt
- **Agent Experience** (0-1) -- structured errors, pagination, versioning, onboarding friction

Each finding is tagged `FOUND`, `PARTIAL`, or `MISSING` with prioritized recommendations.

## Requirements

- Claude Code
- `gh` CLI (only for remote repo scanning)

## Team

Built for Agent Skill Wars. See `Agents-First-Software-Vision.md` at repo root for full design rationale.
