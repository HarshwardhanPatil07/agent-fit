# Sample Report: morning-ops

```
AGENT READINESS REPORT: morning-ops
====================================
Overall Score: 4/10 (Minimally accessible)

MACHINE-READABLE INTERFACES          1/3
  [MISSING] No REST API or OpenAPI spec found
  [MISSING] No MCP server configuration
  [FOUND]   CLI scripts in cron-jobs/ (but no structured output)
  [MISSING] No webhook/event system

AUTHENTICATION MODEL                  1/3
  [MISSING] No auth middleware detected
  [MISSING] No API key or token system
  [N/A]     No API to authenticate against

DOCUMENTATION FOR AGENTS              1/2
  [PARTIAL] README exists but written for humans, not agents
  [MISSING] No AGENTS.md file
  [MISSING] No structured API documentation

DISCOVERABILITY + AGENT EXPERIENCE    1/2
  [MISSING] No robots.txt or sitemap
  [MISSING] No MCP Server Card
  [FOUND]   Plugin manifests in .claude-plugin/ (agent-discoverable)

TOP 5 RECOMMENDATIONS (priority order):
1. Add an API layer (REST or GraphQL) with OpenAPI spec
2. Create AGENTS.md describing how agents should interact with this software
3. Add MCP server configuration so agents can discover capabilities
4. Implement API key auth for machine-to-machine access
5. Add structured JSON output to CLI scripts
```
