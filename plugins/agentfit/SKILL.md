# Agent-Fit Skill

## Description

Analyzes a codebase for agent-readiness and produces a structured report scoring it across 5 dimensions. Works like a linter for agent compatibility -- run it, fix what it finds, ship agent-ready software.

## Usage

```
/agentfit
```

Run in any project directory. Zero config. Read-only analysis -- no network requests, no code modifications.

## Assessment Dimensions

### 1. Machine-Readable Interfaces (0-3 points)
- REST/GraphQL API with OpenAPI/Swagger spec
- MCP server with `.well-known/mcp.json` and tool definitions
- CLI with structured output (JSON) and parseable `--help`
- Webhooks/event subscriptions
- SDK/client libraries (auto-generated or hand-maintained)

### 2. Authentication Model (0-3 points)
- Machine-to-machine auth (API keys, OAuth `client_credentials`, service accounts)
- Human-only auth barriers (CAPTCHAs, email verification, consent screens)
- Programmatic credential management (create/rotate/revoke via API)
- Token lifecycle documentation (expiry, refresh, error responses)

### 3. Documentation for Agents (0-2 points)
- Machine-parseable docs (OpenAPI spec, structured API reference, AGENTS.md)
- Autonomous onboarding (agent can sign up, auth, and call API from docs alone)
- Error code documentation
- Rate limit documentation with headers (`X-RateLimit-*`)

### 4. Discoverability (0-2 points)
- `robots.txt` / `sitemap.xml` in repo
- API catalog or capabilities manifest
- MCP Server Card (`.well-known/mcp.json`)
- `AGENTS.md` file

### 5. Agent Experience (included in Discoverability score)
- Steps to first API call
- Machine-parseable errors with action codes
- Idempotent write operations
- Cursor-based pagination
- API versioning

## Scoring Rubric (0-10)

| Score | Level | Meaning |
|-------|-------|---------|
| 0-2   | Not agent-ready | No programmatic access. Human-only. |
| 3-4   | Minimally accessible | API exists but poorly documented, auth requires human. |
| 5-6   | Partially ready | Decent API, some docs, agent auth available but not default. |
| 7-8   | Agent-friendly | Good API, OpenAPI spec, agent auth, documented errors/limits. |
| 9-10  | Agent-first | Full MCP, CLI, machine auth, AGENTS.md, structured errors. |

## How It Works

1. **Scan** -- reads codebase files against the 5 dimensions
2. **Score** -- evaluates each criterion as FOUND / PARTIAL / MISSING
3. **Report** -- produces structured output with per-dimension scores and prioritized recommendations

## Constraints

- All checks are static (file/code analysis only)
- No network requests to external services
- No code modifications (read-only)
- No external dependencies beyond the Claude Code runtime
