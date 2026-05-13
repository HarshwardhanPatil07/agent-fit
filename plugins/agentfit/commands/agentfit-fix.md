---
description: Generates files to fix missing agent-fit criteria based on the last /agentfit assessment report
---

When invoked, perform the following remediation. This command WRITES files to the target codebase — it generates missing configuration and documentation files based on the last `/agentfit` assessment.

## Arguments

Optional: `--top N` — limit remediation to the N highest-priority missing criteria (default: all remediable criteria).

## Step 1: Locate Assessment Report

1. Extract project name from the current directory name.
2. Look for the most recent JSON sidecar at `/tmp/agentfit-report-{project_name}*.json` (glob for any git SHA suffix).
3. If no JSON file found, ERROR: "No agentfit report found. Run /agentfit first to generate an assessment."
4. Parse the JSON file. Extract: `criteria` (with status and pillar), `metadata` (project_types, skill_version), and `scores` (maturity_level).

## Step 2: Identify Remediable Criteria

From the JSON report, select all criteria with `status: "missing"`. Filter to only those with known remediation templates (listed below).

Prioritize by maturity level (L1 gaps first, then L2, L3, etc.). Within a level, prioritize by impact tier (high first).

If `--top N` was specified, keep only the top N criteria after prioritization.

## Step 3: Generate Remediation Files

For each selected missing criterion, generate the appropriate file using these templates. Adapt content based on the detected project language and type from the JSON metadata.

### Remediation Templates

**agents_md** (L2, Documentation) → Generate `AGENTS.md`:
- Extract project name, language, and build commands from README.md and package manifests
- Include sections: Project Overview, Build Commands, Test Commands, Coding Conventions, Project Structure
- Use language-appropriate commands (e.g., `go build ./...` for Go, `npm run build` for Node.js)

**pre_commit_hooks** (L1, Style & Validation) → Generate `.pre-commit-config.yaml`:
- Detect language and include appropriate hooks:
  - Go: `golangci-lint`, `gofumpt`
  - Python: `ruff`, `mypy`, `black`
  - TypeScript/JavaScript: `eslint`, `prettier`
  - Rust: `cargo fmt`, `cargo clippy`
- Include universal hooks: `trailing-whitespace`, `end-of-file-fixer`, `check-yaml`, `check-added-large-files`

**devcontainer** (L2, Development Environment) → Generate `.devcontainer/devcontainer.json`:
- Select base image from detected language (e.g., `mcr.microsoft.com/devcontainers/go`, `python:3.12`, `node:20`)
- Include relevant VS Code extensions for the language
- Add `postCreateCommand` with the project's setup command (from README or Makefile)

**env_template** (L2, Development Environment) → Generate `.env.example`:
- Scan for environment variable references in the codebase (e.g., `os.Getenv`, `process.env`, `os.environ`)
- Create `.env.example` with discovered variable names and placeholder values
- Add comments explaining each variable's purpose where inferable

**dependency_update_automation** (L3, Security & Governance) → Generate `.github/dependabot.yml`:
- Detect package ecosystem from project language:
  - Go: `gomod`
  - Python: `pip`
  - TypeScript/JavaScript: `npm`
  - Rust: `cargo`
- Set schedule to weekly, target branch to `main`
- Include `github-actions` ecosystem for CI workflow updates

**codeowners** (L2, Security & Governance) → Generate `.github/CODEOWNERS`:
- Analyze `git log --format="%aN" --since="6 months ago" | sort | uniq -c | sort -rn | head -5` to find top contributors
- Create CODEOWNERS with `* @{top_contributor}` as default
- Add language-specific patterns if multiple contributors specialize in different areas

**issue_templates** (L2, Task Discovery) → Generate `.github/ISSUE_TEMPLATE/`:
- Create `bug_report.md` with: Description, Steps to Reproduce, Expected Behavior, Actual Behavior, Environment
- Create `feature_request.md` with: Problem Statement, Proposed Solution, Alternatives Considered, Additional Context

## Step 4: Interactive Output

After generating all files, present the user with a summary:

```
Agent-Fit Fix: Generated {N} files for {N} missing criteria

Files generated:
  - AGENTS.md (agents_md, L2)
  - .pre-commit-config.yaml (pre_commit_hooks, L1)
  - .devcontainer/devcontainer.json (devcontainer, L2)
  ...

Choose an action:
  (a) Commit to current branch
  (b) Create new branch agentfit-fix/{date} and commit
  (c) Leave files unstaged for manual review
```

Wait for the user's choice:
- **(a)**: Stage all generated files and commit with message: `fix: add missing agent-fit criteria files ({list of criteria names})`
- **(b)**: Create branch `agentfit-fix/{YYYY-MM-DD}`, switch to it, stage and commit with the same message
- **(c)**: Do nothing — files are already on disk for the user to review with `git diff` or `git status`

## Step 5: Report

Print a summary of what was generated and the expected impact:

```
Remediation complete.
  Criteria fixed: {N}
  Expected pass rate change: {old}% → ~{estimated_new}%
  Run /agentfit again to verify.
```
