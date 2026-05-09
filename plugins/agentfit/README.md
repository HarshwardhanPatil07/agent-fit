# Agent-Fit: Agent-Readiness Assessment Skill

A Claude Code skill that analyzes your codebase and tells you how ready your software is for AI agents.

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

## Run

Inside Claude Code, in any project directory:

```
/agentfit
```

## What You Get

A structured report scoring your project 0-10 across:

- **Machine-Readable Interfaces** -- APIs, MCP, CLI, webhooks, SDKs
- **Authentication Model** -- machine-to-machine auth, credential management
- **Documentation for Agents** -- OpenAPI, AGENTS.md, error/rate-limit docs
- **Discoverability + Agent Experience** -- robots.txt, API catalog, idempotency, versioning

Each finding is tagged `FOUND`, `PARTIAL`, or `MISSING` with prioritized recommendations.

## Requirements

- Claude Code (that's it -- no dependencies, no API keys, no installs)

## Team

Built for Agent Skill Wars. See `Agents-First-Software-Vision.md` at repo root for full design rationale.
