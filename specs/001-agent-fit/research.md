# Research: Agent-Fit Assessment Skill

**Date**: 2026-05-11
**Feature**: 001-agent-fit

## R1: Skill Architecture — Single Command File vs Multi-File

**Decision**: Single command file (`agentfit.md`)

**Rationale**: Claude Code skills are markdown files that instruct Claude what to do when invoked. The entire assessment logic — all 9 pillars, 75+ criteria definitions, scoring rules, and report template — fits in a single well-structured markdown file. This is how the existing prototype works and aligns with the constitution's simplicity principle.

**Alternatives considered**:
- Multi-file split (one file per pillar): Adds complexity without benefit. Claude reads the entire command file on invocation regardless of split. No performance gain, harder to maintain consistency.
- External config files (YAML/JSON for criteria definitions): Adds a parsing layer and a second format to maintain. The markdown file IS the config.

## R2: Criterion Evaluation Approach

**Decision**: Claude Code reads files, checks patterns, and evaluates criteria using its native tools (Read, Bash for `find`/`grep`/`gh`)

**Rationale**: Each criterion maps to one of these signal types:
1. **File existence**: `find` or `ls` for specific files (`.pre-commit-config.yaml`, `AGENTS.md`, `.devcontainer/`)
2. **Config parsing**: Read file and check for specific settings (mypy strict, ESLint rules, golangci-lint linters)
3. **CI workflow analysis**: Read `.github/workflows/*.yml` and check for specific steps/actions
4. **Dependency check**: Read `package.json`, `go.mod`, `Cargo.toml`, `pyproject.toml` for specific packages
5. **Code search**: `grep` for patterns (health check endpoints, structured logging, circuit breaker imports)
6. **API check**: `gh api` for GitHub settings (branch protection, labels, secret scanning)
7. **Git history**: `git log` for agent co-authorship, release frequency, doc freshness

All of these are achievable with Claude Code's built-in tools. No external dependencies needed.

**Alternatives considered**:
- Custom scripts/executables: Violates zero-dependency constraint
- AST parsing: Overkill for binary criteria; pattern matching sufficient

## R3: HTML Report Generation

**Decision**: Claude generates an HTML file with inline CSS, writes to a temp file, and opens in browser

**Rationale**: The sample report screenshots show a polished dark-themed web UI. Claude can generate HTML with inline styles (no external CSS/JS dependencies). The report is written to a temporary file and opened with the system's default browser. This keeps the skill zero-dependency while producing the visual quality shown in the screenshots.

**Key HTML structure** (derived from screenshots):
1. Header: project name (large), language badge, repo path, pass rate percentage
2. Level progress bar: L1 through L5 with colored segments and percentage labels
3. Narrative summary: headline (e.g., "Strong Testing"), description paragraph
4. Two-column layout: STRENGTHS (numbered, green) + OPPORTUNITIES (numbered, orange)
5. ALL CRITERIA section: per-pillar groups with header showing pass count and percentage
6. Per-criterion rows: status icon (✓/✗/—), name (snake_case), score (1/1, 0/1, —/—), evidence text

**Alternatives considered**:
- Plain text stdout: Less polished, no progress bar visualization, harder to scan
- Markdown file: Cannot render the progress bar or colored indicators natively
- External template engine: Adds dependency

## R4: Scoring Methodology

**Decision**: Pass rate percentage (0-100%) with gated maturity levels

**Rationale**:
- **Overall score**: `(criteria_passed / applicable_criteria) * 100` — shown as "PASS RATE XX%"
- **Per-pillar score**: Same formula scoped to each pillar — shown as "X/Y (ZZ%)"
- **Maturity level**: 1-5, gated. To unlock level N, must pass 80% of level N criteria AND all previous levels
- **Skipped criteria**: Excluded from both numerator and denominator

This matches the benchmark data (CockroachDB 74%, Temporal 74%, etc.) and the sample report format.

**Alternatives considered**:
- Weighted scoring (some criteria worth more): Adds complexity, harder to explain, not in the benchmark data
- 0-10 scale: Less granular, doesn't match the sample report format which shows percentage

## R5: Language Detection Strategy

**Decision**: Detect primary language from project files, apply language-specific criterion evaluation

**Rationale**: Different languages have different tools and conventions:
- **Go**: `go.mod`, `*_test.go`, `internal/`, golangci-lint, gofumpt
- **Python**: `pyproject.toml`/`setup.py`, `test_*.py`, mypy, Black/Ruff
- **TypeScript/JS**: `package.json`/`tsconfig.json`, `*.test.ts`, ESLint, Prettier
- **Rust**: `Cargo.toml`, `#[test]`, Clippy, rustfmt
- **C++**: `CMakeLists.txt`/`Makefile`, clang-format, clang-tidy
- **Swift**: `Package.swift`, XCTest, SwiftLint

Detection order: check for language-specific manifest files at repo root.

**Alternatives considered**:
- GitHub Linguist API: Requires network; violates constraints for most criteria
- File extension counting: Fragile for polyglot repos

## R6: Criterion-to-Level Mapping

**Decision**: Map each criterion to a maturity level based on the framework's progression model

**Rationale**: From the benchmark data:
- **Level 1 (Functional)**: README, linter, type checker, unit tests, formatter — universally passed (>90%)
- **Level 2 (Documented)**: AGENTS.md, devcontainer, pre-commit hooks, branch protection, deps pinned, build docs
- **Level 3 (Standardized)**: Integration tests, secret scanning, distributed tracing, metrics, test coverage thresholds, test isolation, CI feedback speed
- **Level 4 (Optimized)**: Deployment frequency, flaky test detection, build performance, release automation, code quality metrics, feature flags
- **Level 5 (Autonomous)**: Experiment infrastructure, error-to-insight pipeline, progressive rollout, rollback automation — universally failed (0%)

This mapping determines the gated progression and remediation prioritization.

## R7: Determinism / Grounding Strategy

**Decision**: Structured evaluation with explicit criterion definitions prevents non-determinism

**Rationale**: The framework document reports that grounding evaluations on previous reports reduced variance from 7% to 0.6%. For the skill, determinism is achieved by:
1. Each criterion has a binary, unambiguous check (file exists or doesn't, config has setting or doesn't)
2. The skill definition specifies exact files/patterns to check — no subjective judgment
3. Evidence is captured verbatim from file contents — no interpretation

The main non-determinism risk is in the narrative summary and strengths/opportunities selection, which involve some judgment. Mitigate by defining selection rules: top 3 by pillar score for strengths, bottom 3 by pillar score for opportunities.

**Alternatives considered**:
- Storing previous reports for comparison: Adds state management, violates simplicity
- Running multiple times and averaging: Wasteful, slow
