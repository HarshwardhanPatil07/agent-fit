# Design: Agent-Fit

## Problem Statement

Do you think next people on internet will be AI agents and now is the time to make something agents want?

Agents are already browsing the web, doing research, making purchases, much more, but they are doing it on top of software that was designed for humans clicking buttons in a browser.

Agents needs completely new things “Machine-readable interfaces” like APIs, MCPs, and CLI. Agents also needs thorough documentation to enable them for discover, sign up for, and instantly start using new tools programmatically, without needing a human in the loop.

That means every major category of software that people use today needs to be rebuilt for agents and the new agent first software.

So while everyone is building a skill to do a task, what if we build the skill which assesses the software if it is agent-first?


## Demand Evidence

- Founder's own pain: building agent automations and hitting walls because the target software wasn't designed for agents
- Others online expressing frustration with software lacking agent-friendly interfaces
- MCP has 97M monthly SDK downloads and 10K+ servers -- the ecosystem is growing fast but most software hasn't caught up
- Cloudflare's isitagentready.com exists (scores websites on agent readiness) but only checks web-layer signals, not your actual codebase


## Status Quo

Developers building software today have no systematic way to know if their product is agent-ready. They ship APIs without thinking about machine discoverability. They use OAuth flows that require human consent screens. They write docs for humans, not for LLMs. They don't expose MCP servers or CLI tools.

The result: agent developers trying to integrate with their software hit walls. They build fragile workarounds, hardcode credentials, scrape UIs. The software creator never knows their product is losing agent-driven adoption because there's no feedback loop.

## Target User and Narrowest Wedge

**Target user:** Developers and teams building software (SaaS, APIs, platforms, tools, libraries) who want their product to be usable by AI agents. The person who asks: "Is my software agent-ready? What am I missing?"

**Narrowest wedge:** A Claude Code agent skill that analyzes your own codebase and project, then produces an agent-readiness report with specific gaps and recommendations. Run it like a linter. Fix what it finds. Ship agent-ready software.

## What This Skill Assesses (The Agent-Readiness Dimensions)

The skill examines your project across these dimensions:

### 1. Machine-Readable Interfaces
- **REST/GraphQL API:** Does it exist? Is there an OpenAPI/Swagger spec? Are endpoints documented with request/response schemas?
- **MCP Server:** Is there an MCP server? Is there metadata at /.well-known/mcp.json? Are tools properly defined with descriptions and input schemas?
- **CLI:** Is there a command-line interface? Does it support structured output (JSON)?  Does it have --help with parseable output?
- **Webhooks/Events:** Can agents subscribe to events programmatically?
- **SDK/Client Libraries:** Do they exist? Are they auto-generated from the API spec or hand-maintained?

### 2. Authentication Model
- **Agent-friendly auth:** Does the product support machine-to-machine auth? (API keys, OAuth client_credentials, service accounts)
- **Human-only auth:** Does signup/login require human intervention? (CAPTCHAs, email verification, OAuth authorization_code with consent screens, MFA)
- **Programmatic credential management:** Can API keys be created, rotated, and revoked via API? Or only through a web UI?
- **Token lifecycle:** Are token expiry and refresh mechanisms documented? Do credentials fail silently or with clear error responses?

### 3. Documentation for Agents
- **Machine-parseable docs:** Is there an OpenAPI spec, API reference with structured examples, or AGENTS.md?
- **Onboarding without a human:** Can an agent figure out how to sign up, authenticate, and make its first API call by reading the docs alone?
- **Error documentation:** Are error codes and responses documented so agents can handle failures programmatically?
- **Rate limit documentation:** Are limits documented with headers (X-RateLimit-*) so agents can self-throttle?

### 4. Discoverability (static checks on repo files)
- **robots.txt / sitemap:** Is there a robots.txt or sitemap file in the repo (e.g., public/robots.txt, static/sitemap.xml)?
- **API catalog:** Is there a published list of capabilities in the repo (e.g., api-catalog.json, capabilities manifest)?
- **MCP Server Card:** Is there a .well-known/mcp.json file in the repo?
- **AGENTS.md:** Does the repo include an AGENTS.md file describing how agents should interact with this software?

### 5. Agent Experience (AX)
- **Time to first API call:** How many steps from "I found this software" to "I made a successful API request"?
- **Error quality:** Are errors machine-parseable with action codes, not just human-readable strings?
- **Idempotency:** Are write operations idempotent (safe to retry without side effects)?
- **Pagination:** Are list endpoints paginated with cursor-based navigation?
- **Versioning:** Is the API versioned so agents don't break on updates?

## Scoring Rubric (0-10)

| Score | Level | What it means |
|-------|-------|---------------|
| 0-2   | **Not agent-ready** | No API, no programmatic access. Human-only product. Agents cannot use this. |
| 3-4   | **Minimally accessible** | API exists but poorly documented, auth requires human, no MCP/CLI. Agents can technically use it but with significant friction. |
| 5-6   | **Partially ready** | Decent API with some docs, agent-friendly auth available but not default, missing MCP or CLI. Agents can use it with workarounds. |
| 7-8   | **Agent-friendly** | Good API coverage, OpenAPI spec, agent-friendly auth, documented errors and rate limits. Missing MCP server or some polish. |
| 9-10  | **Agent-first** | Full agent-first design: MCP server, CLI, client_credentials/API keys, programmatic key management, complete API coverage, AGENTS.md, structured errors, idempotent operations. Agents are first-class users. |

Sub-scores (each contributes to overall):
- Machine-readable interfaces: 0-3 points
- Authentication model: 0-3 points
- Documentation for agents: 0-2 points
- Discoverability + Agent experience: 0-2 points

## How the Skill Works

### Invocation

```
/agentfit
```

Run it in your project directory. The skill analyzes everything it can find.

### What the Skill Does

1. **Scans the codebase** against the 5 dimensions defined above (machine-readable interfaces, auth model, documentation, discoverability, agent experience)
2. **Reads files and code** to check for each dimension's criteria -- all checks are static, no network requests
3. **Produces a structured report** with overall score, per-dimension scores, specific findings (FOUND/PARTIAL/MISSING), and prioritized recommendations

### What the Skill Does NOT Do

- Does NOT browse external websites or crawl login pages
- Does NOT make network requests to production systems
- Does NOT modify any code (read-only analysis)
- Does NOT require any external dependencies beyond Claude Code

### Output Example

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

## Presentation Context: Agent Skill Wars



### Jury Q&A Prep

| Question | Answer |
|----------|--------|
| **Usability** | Run `/agentfit` in any project. Zero config. Reads your codebase and produces a report. |
| **Clarity** | Five dimensions, each scored. Overall 0-10 with a rubric. Color-coded findings: FOUND, PARTIAL, MISSING. |
| **Practical value** | Before shipping, know if agents can use your software. Fix the gaps BEFORE agent developers hit walls. |
| **Reproducibility** | Copy one SKILL.md file. That's it. No dependencies, no installs, no accounts. |
| **Risks** | Static analysis can miss runtime behavior. Scoring rubric is opinionated. Agent-readiness is a new concept without industry consensus. |


## Skill Structure for the Repo

```
plugins/agentfit/
  SKILL.md              # The skill definition
  README.md             # How to install and use
  commands/
    assess.md           # The /agentgate command spec
  examples/
    sample-report.md    # Example output for reference
```

### Transferability

To adopt this skill:
1. Copy `plugins/agentfit/` to your `.claude/skills/agentfit/` directory
2. Run `/agentfit` in any project
3. No dependencies. No API keys needed (uses the Claude instance you're already running in).

## Competitive Moat

### Why this is different from Cloudflare and Factory.ai

| | Cloudflare isitagentready.com | Factory.ai /readiness-report | agentgate |
|---|---|---|---|
| **What it assesses** | Websites (HTTP layer) | Your codebase (dev workflow) | Your software's agent-consumability |
| **How it works** | Checks robots.txt, sitemaps, MCP cards | Checks linters, tests, CI, docs | Analyzes APIs, auth, docs, MCP, CLI |
| **Who it's for** | Website owners | Developers (code quality) | Developers (agent compatibility) |
| **What it misses** | Auth flows, API depth, docs quality | Whether agents can use the software | Runtime behavior, live API testing |
| **Distribution** | Web tool (free) | CLI (integrated in Factory) | Claude Code skill (copy SKILL.md) |

### Defensible position
- **Expertise encoding:** The scoring rubric and dimension framework IS the product. It encodes a specific point of view on what agent-ready means.
- **Community adoption:** If developers start running this and sharing scores, the rubric becomes a standard.
- **Temporal tracking:** Run it before and after changes. Track your agent-readiness score over time. Include it in CI.

## Open Questions

1. Should the skill also generate fix PRs (like Factory.ai does) or stay read-only for v1?
2. What's the right default for the scoring weights? Should users be able to customize?
3. Should there be a "quick" mode (just the score) vs. "full" mode (detailed report)?

## Success Criteria

- Successful live demo at Agent Skill Wars
- At least 3 audience members try the skill on their own projects after the event
- Feedback from jury validates practical value and reproducibility
- Skill produces useful, non-obvious findings on at least 3 different project types

## The Assignment

1. Build the SKILL.md file in this repo's `plugins/agentfit/` directory
2. Test it on agent-fit (should score low -- no API, no MCP)
3. Test it on at least 2 other repos you have access to
4. Prepare the 5-minute demo script
5. Practice the demo once to calibrate timing


