---
description: Analyzes a codebase for agent-readiness and produces a structured report scoring it across 5 dimensions
---

When invoked, perform the following steps in order:

### Step 1: Discover Project Structure
- List root directory files and folders
- Identify project type (Node.js, Python, Go, Ruby, Rust, Java, etc.)
- Locate key config files (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, etc.)

### Step 2: Check Machine-Readable Interfaces
- Search for OpenAPI/Swagger specs (`openapi.yaml`, `openapi.json`, `swagger.*`)
- Search for MCP config (`.well-known/mcp.json`, `mcp.json`, MCP server files)
- Search for CLI entry points (`bin/`, `cli.*`, `commands/`)
- Search for webhook/event definitions
- Search for SDK directories or auto-generated client code

### Step 3: Check Authentication Model
- Search for auth middleware/modules (`auth/`, `middleware/auth.*`, `passport.*`, etc.)
- Check for API key patterns in config or env files
- Check for OAuth configuration (client_credentials vs authorization_code)
- Check for CAPTCHA or human-gated flows in signup/login code

### Step 4: Check Documentation
- Check for OpenAPI spec completeness (schemas, examples, descriptions)
- Check for `AGENTS.md`
- Check for structured error documentation
- Check for rate limit headers in API response code
- Evaluate if docs enable autonomous onboarding

### Step 5: Check Discoverability
- Check for `robots.txt`, `sitemap.xml` in static/public directories
- Check for API catalog or capabilities manifest
- Check for `.well-known/mcp.json`
- Check for `AGENTS.md`

### Step 6: Evaluate Agent Experience
- Count steps to first API call based on docs
- Check error response format (structured JSON vs plain strings)
- Check for idempotency keys/headers in write endpoints
- Check for cursor-based pagination
- Check for API versioning in routes or headers

### Step 7: Score and Report
- Calculate per-dimension scores
- Calculate overall score (0-10)
- Map to rubric level
- Generate top 5 prioritized recommendations
- Output the structured report

## Output Format

```
AGENT READINESS REPORT: <project-name>
====================================
Overall Score: X/10 (<level>)

MACHINE-READABLE INTERFACES          X/3
  [FOUND|PARTIAL|MISSING] <finding>
  ...

AUTHENTICATION MODEL                  X/3
  [FOUND|PARTIAL|MISSING] <finding>
  ...

DOCUMENTATION FOR AGENTS              X/2
  [FOUND|PARTIAL|MISSING] <finding>
  ...

DISCOVERABILITY + AGENT EXPERIENCE    X/2
  [FOUND|PARTIAL|MISSING] <finding>
  ...

TOP 5 RECOMMENDATIONS (priority order):
1. ...
2. ...
3. ...
4. ...
5. ...
```
