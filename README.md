# Agent-Fit

Do you think next people on internet will be AI agents and now is the time to make something agents want?

Agents are already browsing the web, doing research, making purchases, much more, but they are doing it on top of software that was designed for humans clicking buttons in a browser.

Agents needs completely new things "Machine-readable interfaces" like APIs, MCPs, and CLI. Agents also needs thorough documentation to enable them for discover, sign up for, and instantly start using new tools programmatically, without needing a human in the loop.

That means every major category of software that people use today needs to be rebuilt for agents and the new agent first software.

So while everyone is building a skill to do a task, what if we build the skill which assesses the software if it is agent-first?

## What's Inside

| Plugin | Description | Docs |
|--------|-------------|------|
| agentfit | Agent-readiness assessment -- scores your codebase 0-10 across 5 dimensions | [README](plugins/agentfit/README.md) |

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

No cloning, no copying files. Claude Code fetches the plugin automatically.

## Usage

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

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Design Rationale

See [Agents-First-Software-Vision.md](Agents-First-Software-Vision.md) for the full design document.
