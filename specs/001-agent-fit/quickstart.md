# Quickstart: Agent-Fit

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

In any project directory within Claude Code:

```
/agentfit
```

## What Happens

1. The skill detects your project's primary language
2. Scans the codebase against 75+ criteria across 9 pillars
3. Generates an HTML report and opens it in your browser
4. Report shows: overall pass rate, maturity level (L1-L5), per-pillar scores, per-criterion evidence, top strengths and opportunities

## Time to First Report

Under 2 minutes for projects up to 10,000 files. No setup, no API keys, no dependencies beyond Claude Code.
