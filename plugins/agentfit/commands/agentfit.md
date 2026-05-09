---
description: Scans a local or remote codebase and extracts agent-fit signals across 5 dimensions into a structured report
---

When invoked, follow these phases in order. The goal is to scan the target software and extract agent-fit signals — how easily AI agents can discover, authenticate with, and programmatically interact with it.

`$ARGUMENTS` contains the optional target (empty = scan current directory).

---

## Phase 0: Determine Scan Target

Parse `$ARGUMENTS` to determine what to scan:

- **Empty**: Local mode. Set `SCAN_DIR` to the current working directory. Try to extract repo name from `git remote get-url origin` (use the repo name portion). If not a git repo, use the directory name.
- **`owner/repo` format** (e.g., `expressjs/express`): Remote mode.
- **GitHub URL** (e.g., `https://github.com/owner/repo`): Remote mode. Extract `owner/repo` from the URL.

**Remote mode setup:**
1. Verify `gh` CLI is available: `which gh`. If missing, stop and tell the user to install it.
2. Verify authentication: `gh auth status`. If not authenticated, stop and tell the user to run `gh auth login`.
3. Create temp directory: `SCAN_DIR=$(mktemp -d /tmp/agentfit-XXXXXX)`
4. Shallow clone: `gh repo clone {owner}/{repo} "$SCAN_DIR" -- --depth 1 --single-branch`
5. If clone fails, report the error, clean up the temp directory, and stop.
6. Fetch repo metadata: `gh api repos/{owner}/{repo} --jq '{stargazers_count, forks_count, language, visibility, default_branch}'`

Tell the user what you're scanning and whether it's local or remote.

---

## Phase 1: Discover Project

Run these commands in `$SCAN_DIR`:

1. **File tree**: `find "$SCAN_DIR" -type f -not -path '*/.git/*' -not -path '*/node_modules/*' -not -path '*/vendor/*' -not -path '*/__pycache__/*' -not -path '*/dist/*' -not -path '*/build/*' | head -2000`
2. **Project type**: Check for config files to determine the ecosystem:
   - `package.json` → Node.js (check for `next`, `express`, `fastify`, `nestjs` in dependencies)
   - `pyproject.toml` / `setup.py` / `requirements.txt` → Python (check for `flask`, `django`, `fastapi`)
   - `go.mod` → Go
   - `Cargo.toml` → Rust
   - `pom.xml` / `build.gradle` → Java
   - `Gemfile` → Ruby
3. **Git metadata**: `git -C "$SCAN_DIR" log --oneline -5 2>/dev/null` to confirm it's a git repo and see recent activity.

Record the project name, type, and primary language for the report header.

---

## Phase 2: Scan Signals

For each signal below, search within `$SCAN_DIR`. Record the status as **FOUND**, **PARTIAL**, or **MISSING**, along with brief evidence (file path or pattern matched). Cap grep/find results to 20 matches per signal to avoid context overflow.

### Dimension 1: Machine-Readable Interfaces

1. **REST API endpoints**: Search for route definitions — `app.get(`, `app.post(`, `router.`, `@app.route`, `@api_view`, `HandleFunc`, `#[get(`, controller files in `controllers/`, `routes/`, `api/`. FOUND = route files with multiple endpoints. PARTIAL = few endpoints or no clear structure. MISSING = no routes found.

2. **OpenAPI/Swagger spec**: Search for `openapi.yaml`, `openapi.json`, `swagger.yaml`, `swagger.json`, `api-spec.*`. If found, check for `paths` and `components` or `definitions` keys. FOUND = spec with schemas. PARTIAL = spec exists but incomplete. MISSING = no spec.

3. **MCP server**: Search for `.well-known/mcp.json`, `mcp.json`, files containing `McpServer`, `mcp_server`, `@mcp`, or MCP tool definitions. FOUND = MCP server with tool definitions. PARTIAL = MCP config without tools. MISSING = none.

4. **CLI entry points**: Check `bin/` directory, `"bin"` field in package.json, files named `cli.*`, `main.go` with `cobra`/`flag`, `argparse`/`click`/`typer` imports in Python. Check if CLI supports `--json` or structured output. FOUND = CLI with structured output. PARTIAL = CLI without structured output. MISSING = no CLI.

5. **Webhooks/Events**: Search for webhook handler files, event schema definitions, `pubsub`, `EventEmitter` exports, `@webhook`, `webhook_url`, event subscription endpoints. FOUND = webhook/event system. MISSING = none found.

6. **SDK/Client libraries**: Search for `clients/`, `sdk/`, auto-generated client code, or published packages that are client libraries. FOUND = SDK exists. MISSING = none.

7. **GraphQL schema**: Search for `schema.graphql`, `*.gql`, `*.graphqls`, `typeDefs`, `@ObjectType`, `type Query`. FOUND = GraphQL schema with types. MISSING = none.

8. **gRPC/Protobuf**: Search for `*.proto` files with `service` definitions. FOUND = proto service definitions. MISSING = none.

### Dimension 2: Authentication Model

1. **Auth middleware**: Search for files in `auth/`, `middleware/auth.*`, `passport.*`, `@auth`, `@login_required`, `authorize`, JWT verification code, `jsonwebtoken`, `PyJWT`. FOUND = auth system present. MISSING = none.

2. **API key support**: Search for API key handling — `x-api-key`, `apiKey`, `API_KEY`, API key validation middleware, key generation endpoints. FOUND = API key auth supported. MISSING = none.

3. **OAuth client_credentials**: Search for `client_credentials`, `client_id.*client_secret`, machine-to-machine auth configuration. FOUND = M2M auth available. MISSING = none.

4. **Human-gated auth (negative)**: Search for CAPTCHA integration (`recaptcha`, `hcaptcha`, `turnstile`), email verification requirements, MFA enforcement on all flows. FOUND = human-gated flows present (reduces score). MISSING = no human gates (good for agents).

5. **Programmatic credential management**: Search for API endpoints to create/rotate/revoke keys or tokens programmatically. FOUND = programmatic key management. MISSING = manual only.

6. **Token lifecycle docs**: Search README, docs/, or inline comments for token expiry, refresh mechanism, or credential error documentation. FOUND = documented. MISSING = undocumented.

### Dimension 3: Documentation for Agents

1. **AGENTS.md or CLAUDE.md**: Search for `AGENTS.md`, `CLAUDE.md`, `.claude/` directory with agent instructions. FOUND = agent-specific docs exist. MISSING = none.

2. **OpenAPI spec completeness**: If an OpenAPI spec was found in Dimension 1, check for: response examples, parameter descriptions, error response schemas. FOUND = complete with examples. PARTIAL = exists but missing examples/descriptions. MISSING = no spec or N/A.

3. **Structured error docs**: Search for error code documentation, error response format docs, or consistent error response types (e.g., `ErrorResponse` class/type). FOUND = error responses documented. MISSING = none.

4. **Rate limit documentation**: Search for `rate.limit`, `X-RateLimit`, `throttle`, `429`, rate limit documentation in README or docs. FOUND = rate limits documented with headers. PARTIAL = rate limiting exists but undocumented. MISSING = none.

5. **Machine-parseable README**: Check if README.md exists and contains: installation commands (code blocks), API usage examples, or quick-start sections an agent could follow. FOUND = README with actionable code blocks. PARTIAL = README exists but narrative only. MISSING = no README.

6. **API schema documentation**: Search for AsyncAPI specs, JSON Schema files, protobuf documentation, or generated API reference docs. FOUND = schema docs exist. MISSING = none.

### Dimension 4: Discoverability

1. **robots.txt / sitemap.xml**: Search for `robots.txt`, `sitemap.xml` in `public/`, `static/`, or root. FOUND = present. MISSING = none.

2. **API catalog / capabilities manifest**: Search for `api-catalog.json`, `capabilities.json`, `.well-known/` directory with discovery files, or an API index endpoint. FOUND = catalog exists. MISSING = none.

3. **MCP Server Card**: Search for `.well-known/mcp.json` specifically. FOUND = MCP card present. MISSING = none.

4. **AGENTS.md presence**: (Cross-check from Dimension 3.) FOUND = present. MISSING = none.

### Dimension 5: Agent Experience

1. **Structured error responses**: Look at error handling code — do errors return JSON with error codes/types, or plain strings? Search for error response patterns, `ErrorResponse`, `error_code`, `status_code` in response objects. FOUND = structured JSON errors. PARTIAL = mix of structured and unstructured. MISSING = plain string errors or none found.

2. **Idempotency support**: Search for `idempotency`, `Idempotency-Key`, `idempotent`, or patterns showing safe retry logic on write endpoints. FOUND = idempotency supported. MISSING = none.

3. **Cursor-based pagination**: Search for `cursor`, `next_cursor`, `after`, `before` in list/query endpoints, or `offset`/`limit` patterns. FOUND = cursor pagination. PARTIAL = offset pagination only. MISSING = no pagination.

4. **API versioning**: Search for `/v1/`, `/v2/`, `api-version` header, version prefix in routes, or versioning documentation. FOUND = versioned API. MISSING = no versioning.

5. **Time-to-first-API-call**: Evaluate based on findings — how many steps from "I found this repo" to "I made a successful API request"? Count: find docs → understand auth → get credentials → make request. FOUND = achievable in ≤3 steps with clear docs. PARTIAL = possible but requires manual setup. MISSING = unclear or impossible without human help.

---

## Phase 3: Score

Calculate scores using these rules:

**Machine-Readable Interfaces (0–3 points):**
- 0 = No programmatic interfaces found
- 1 = One interface type found (e.g., CLI only, or basic routes without spec)
- 2 = API with OpenAPI spec, OR MCP server, OR multiple interface types
- 3 = Multiple interface types with complete specs (e.g., REST + OpenAPI + CLI + webhooks)

**Authentication Model (0–3 points):**
- 0 = No auth system, or only human-gated auth
- 1 = Auth exists but requires human setup (manual key creation, consent screens only)
- 2 = API key or M2M auth available but limited (no programmatic key management)
- 3 = Full M2M auth with programmatic credential management, documented token lifecycle

**Documentation for Agents (0–2 points):**
- 0 = No agent-relevant docs (no README code blocks, no API docs, no AGENTS.md)
- 1 = Some docs exist (README with examples, or partial API docs) but not agent-optimized
- 2 = AGENTS.md present, complete OpenAPI spec with examples, error and rate-limit docs

**Discoverability (0–1 point):**
- 0 = No discovery mechanisms
- 1 = At least one of: MCP card, AGENTS.md, API catalog, robots.txt

**Agent Experience (0–1 point):**
- 0 = Poor AX (unstructured errors, no pagination, no versioning, unclear onboarding)
- 1 = Good AX (structured errors, pagination, versioning, achievable onboarding in ≤3 steps)

**Overall Score** = sum of all dimensions (0–10).

**Level mapping:**

| Score | Level |
|-------|-------|
| 0–2   | Not agent-fit |
| 3–4   | Minimally fit |
| 5–6   | Partially fit |
| 7–8   | Agent-friendly |
| 9–10  | Fully agent-fit |

---

## Phase 4: Report

Output the report in this exact format:

```
AGENT-FIT REPORT: {project-name}
====================================
Scan mode: {local | remote}
{If remote: Stars: X | Forks: X | Language: X}
Overall Score: X/10 ({level})

MACHINE-READABLE INTERFACES          X/3
  [{status}] {finding}
  ...

AUTHENTICATION MODEL                  X/3
  [{status}] {finding}
  ...

DOCUMENTATION FOR AGENTS              X/2
  [{status}] {finding}
  ...

DISCOVERABILITY                       X/1
  [{status}] {finding}
  ...

AGENT EXPERIENCE                      X/1
  [{status}] {finding}
  ...

TOP 5 RECOMMENDATIONS (priority order):
1. ...
2. ...
3. ...
4. ...
5. ...
```

Each finding should be one line with the status tag and a concise description of what was found or is missing. Recommendations should be specific and actionable — not generic advice.

---

## Phase 5: Cleanup

If running in **remote mode**, clean up the cloned repository:
```bash
rm -rf "$SCAN_DIR"
```

Confirm cleanup to the user: "Temporary clone removed."

If running in **local mode**, no cleanup needed.
