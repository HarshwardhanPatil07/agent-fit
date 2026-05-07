# Contributing to Agent-Fit

Thanks for contributing. Keep changes small, clear, and aligned with the project goal: building the `agentfit` skill for agent-readiness assessment.

## Scope (Current Phase)

- Focus only on skill content and documentation improvements.
- Do not add CI/CD setup yet.
- Do not add unrelated framework or infrastructure code.

## Project Structure to Follow

Keep work inside the current structure unless explicitly discussed with the team:

- `plugins/agentfit/.claude-plugin/plugin.json`
- `plugins/agentfit/commands/agentfit.md`
- `plugins/agentfit/README.md`
- `plugins/agentfit/examples/sample-report.md`
- `.claude-plugin/marketplace.json`

The `.claude-plugin/plugin.json` file is the marketplace manifest. Update the `version` field in this file when making a release. Do not change the `name`, `owner`, or `source` fields without team discussion.

## Branch Naming (Required)

Use clear and identifiable branch names so everyone can quickly understand what was pushed.

Format:

`<name>/<short-purpose>`

Examples:

- `shreyash/initial-base-structure`
- `shreyash/update-skill-rubric`
- `riya/improve-assess-command-flow`
- `team/docs-cleanup`

Rules:

- Use lowercase letters.
- Use hyphens (`-`) between words.
- Keep names short but meaningful.
- Avoid generic names like `test`, `update`, `changes`, `new`.

## Commit Message Guidelines

Use concise commit messages with intent first.

Examples:

- `docs: add contribution guidelines`
- `skill: refine scoring rubric language`
- `command: clarify /agentfit output sections`

## Pull Request Guidelines

- One focused PR per change.
- Include a short summary of what changed and why.
- If output format changes, include before/after snippets in the PR description.
- Request at least one teammate review before merge.

## Content Quality Checklist

Before opening a PR, ensure:

- Wording is clear and consistent with the project vision.
- No contradiction between `SKILL.md`, command spec, and sample report.
- Terminology stays consistent (`FOUND`, `PARTIAL`, `MISSING`, score rubric, etc.).
- No unnecessary files or folders were added.

## Quick Start

```bash
git checkout main
git pull
git checkout -b shreyash/initial-base-structure
```

Keep contribution quality high and repository structure clean.
