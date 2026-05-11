# Feature Specification: Agent-Fit Assessment Skill

**Feature Branch**: `001-agent-fit`
**Created**: 2026-05-11
**Status**: Draft
**Input**: User description: "Build a skill that assesses software agent-readiness across 9 pillars, 75+ criteria, and 5 maturity levels"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Run Agent-Fit Assessment (Priority: P1)

A developer runs `/agentfit` in their project directory. The skill scans the codebase against 9 pillars (Style & Validation, Build System, Testing, Documentation, Development Environment, Debugging & Observability, Security & Governance, Task Discovery, Product & Experimentation) containing 75+ binary criteria. It produces a structured report with an overall maturity level (1-5), per-pillar percentage scores, and each criterion tagged FOUND or MISSING.

**Why this priority**: This is the core value proposition — without the assessment, nothing else exists.

**Independent Test**: Run `/agentfit` in any project directory and verify a complete report is produced with maturity level, pillar scores, and per-criterion findings.

**Acceptance Scenarios**:

1. **Given** a project directory with source code, **When** the user runs `/agentfit`, **Then** the skill produces a report with a maturity level (1-5), 9 pillar percentage scores, and at least one finding per pillar tagged FOUND or MISSING.
2. **Given** a project with no linter, no tests, and no README, **When** the user runs `/agentfit`, **Then** the maturity level is 1 (Functional) with specific MISSING findings for each absent criterion.
3. **Given** a project like CockroachDB with comprehensive tooling, **When** the user runs `/agentfit`, **Then** the maturity level is 4 (Optimized) with a score around 74%.

---

### User Story 2 - Understand Per-Criterion Findings (Priority: P2)

A developer has received their report and wants to understand exactly what passed and what failed. Each criterion finding includes what was checked, what signal was used (file existence, config parsing, CI workflow, etc.), and what specific tool or file would satisfy it.

**Why this priority**: The maturity level alone is not actionable. Developers need per-criterion detail to know what to fix.

**Independent Test**: Review report findings and verify each MISSING criterion specifies what to add (e.g., "Add .pre-commit-config.yaml with language-appropriate hooks").

**Acceptance Scenarios**:

1. **Given** a completed report, **When** the developer reads a MISSING finding, **Then** it specifies what is missing and what to create (e.g., "No AGENTS.md file found — create one documenting build commands, test instructions, and project conventions").
2. **Given** a FOUND finding, **When** the developer reads it, **Then** it identifies the specific file or configuration that satisfied the criterion (e.g., "Formatter: Black configured in pyproject.toml").

---

### User Story 3 - Prioritized Remediation Recommendations (Priority: P3)

After seeing the report, the developer wants to know what to fix first for maximum impact on their maturity level. The report includes recommendations ordered by level progression — fix Level 1 gaps before Level 2, Level 2 before Level 3.

**Why this priority**: Developers have limited time. Level-gated progression means fixing Level 4 criteria is wasted effort if Level 2 fundamentals are missing.

**Independent Test**: Verify the report includes a remediation section with items ordered by maturity level progression.

**Acceptance Scenarios**:

1. **Given** a project at Level 2, **When** the report is generated, **Then** recommendations prioritize Level 3 criteria gaps before Level 4 or 5 gaps.
2. **Given** a project missing both a README (Level 1) and distributed tracing (Level 3), **Then** the README is recommended first.

---

### Edge Cases

- What happens when the project directory is empty (no source files)?
- How does the skill handle a monorepo with multiple services?
- What happens when files are binary or non-text?
- How does the skill handle projects in unsupported languages?
- What happens when a criterion is not applicable (e.g., N+1 detection for a database project)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The skill MUST evaluate the codebase against 9 pillars containing 75+ binary (pass/fail) criteria.
- **FR-002**: The skill MUST assign a maturity level from 1 (Functional) to 5 (Autonomous) based on the 80% gated progression rule — a repository MUST pass 80% of criteria at a level AND all previous levels to unlock it.
- **FR-003**: The skill MUST produce per-pillar percentage scores (e.g., "Style & Validation: 73%").
- **FR-004**: The skill MUST tag each criterion as FOUND or MISSING with specific evidence (file path, config value, or absence).
- **FR-005**: The skill MUST support multiple signal types for criterion evaluation: file existence, config parsing, CI workflow analysis, dependency check, code search, and documentation content analysis.
- **FR-006**: The skill MUST support skipping criteria that are not applicable to a repository's nature (e.g., DAST scanning for CLI tools) without counting against the score.
- **FR-007**: The skill MUST produce prioritized remediation recommendations ordered by level progression.
- **FR-008**: The skill MUST be read-only — no modifications to the target codebase, no network requests.
- **FR-009**: The skill MUST require zero external dependencies — only Claude Code is needed.
- **FR-010**: The skill MUST handle edge cases gracefully — empty directories, binary files, unsupported languages — by reporting what it cannot assess.
- **FR-011**: The skill MUST produce deterministic scores — identical codebase state produces identical reports (variance under 1% through grounding on previous reports).
- **FR-012**: The report MUST be rendered as a web-based HTML page with dark theme, containing: project header (name, language badge, repo path, overall pass rate), level progress bar (L1-L5 with completion percentages), narrative summary headline, Strengths column (top 3 passing areas with evidence), Opportunities column (top 3 gaps with remediation), and a full ALL CRITERIA section with per-pillar breakdown showing each criterion's status (✓ pass, ✗ fail, — skipped), score (1/1, 0/1, —/—), and evidence text.
- **FR-013**: Criteria names in the report MUST use snake_case format (e.g., `pre_commit_hooks`, `dead_code_detection`).
- **FR-014**: Skipped criteria MUST show —/— score with an explanation of why the criterion was skipped (e.g., "Skipped - temporal is a database-backed service but uses raw SQL without ORM").
- **FR-015**: The skill MUST use the `gh` CLI for criteria requiring GitHub API access (branch protection, issue labeling system, backlog health, secret scanning) when the user is authenticated. When `gh` is not available or not authenticated, those criteria MUST be gracefully skipped (—/— with explanation), not failed.

### Pillar 1: Style and Validation (13 criteria)

- Formatter configured and enforced
- Lint config with specific rule sets
- Type check enabled and configured
- Strict typing mode enabled
- Pre-commit hooks configured
- Naming consistency documented or enforced
- Large file detection configured
- Code modularization enforcement
- Cyclomatic complexity analysis
- Dead code detection
- Duplicate code detection
- Tech debt tracking (TODO/FIXME scanning)
- N+1 query detection

### Pillar 2: Build System (19 criteria)

- Build commands documented
- Dependencies pinned via lockfile
- VCS CLI tools available
- Single command setup documented
- Fast CI feedback (under 10 minutes)
- Deployment frequency (regular releases)
- Release automation via CI/CD
- Release notes automation
- Automated PR review bots
- Agentic development tooling
- Feature flag infrastructure
- Build performance tracking
- Unused dependency detection
- Monorepo tooling
- Dead feature flag detection
- Heavy dependency detection
- Progressive rollout configuration
- Rollback automation
- Version drift detection

### Pillar 3: Testing (8 criteria)

- Unit tests exist
- Unit tests runnable via documented command
- Integration/E2E tests exist
- Test coverage thresholds enforced
- Test naming conventions followed
- Test isolation configured
- Flaky test detection
- Test performance tracking

### Pillar 4: Documentation (8 criteria)

- README exists and is comprehensive
- AGENTS.md or CLAUDE.md exists
- AGENTS.md validation in CI
- Documentation freshness (updated within 180 days)
- Automated doc generation
- API schema docs (OpenAPI, GraphQL, protobuf)
- Service flow/architecture diagrams
- Agent skills configured

### Pillar 5: Development Environment (5 criteria)

- Devcontainer configuration exists
- Devcontainer builds and runs
- Environment variable template provided
- Local services containerized
- Database schema management exists

### Pillar 6: Debugging and Observability (11 criteria)

- Structured logging configured
- Distributed tracing instrumented
- Metrics collection configured
- Alerting configured
- Deployment observability dashboards
- Error tracking with release context
- Health check endpoints
- Profiling instrumentation
- Code quality metrics tracked
- Circuit breakers implemented
- Runbooks documented

### Pillar 7: Security and Governance (11 criteria)

- Branch protection enabled
- CODEOWNERS defined
- Secret scanning configured
- Secrets management (vault/manager)
- Dependency update automation
- Comprehensive .gitignore
- Automated security review in CI
- Log scrubbing for sensitive data
- PII handling and classification
- DAST scanning
- Privacy compliance enforcement

### Pillar 8: Task Discovery (4 criteria)

- Issue templates exist
- Issue labeling system configured
- Backlog health (titles >10 chars, >70% labeled)
- PR templates exist

### Pillar 9: Product and Experimentation (3 criteria)

- Product analytics instrumentation
- Error-to-insight pipeline
- Experiment infrastructure (A/B testing)

### Key Entities

- **Assessment Report**: Structured output containing maturity level, pillar scores, per-criterion findings, and remediation recommendations.
- **Pillar**: One of 9 assessment categories, each containing multiple binary criteria with defined signal types.
- **Criterion**: A single binary (pass/fail) check with a signal type (file existence, config parsing, CI workflow, etc.) and specific tools/files that satisfy it.
- **Maturity Level**: One of 5 levels (Functional, Documented, Standardized, Optimized, Autonomous) determined by gated 80% progression.
- **Finding**: A criterion result tagged FOUND or MISSING with evidence and remediation guidance.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can run the assessment and receive a complete report in under 2 minutes for projects up to 10,000 files.
- **SC-002**: The skill produces useful, non-obvious findings on at least 3 different project types (Go service, Python library, TypeScript web app).
- **SC-003**: Every MISSING finding includes a specific, actionable remediation with tool/file examples.
- **SC-004**: Scores are deterministic — identical codebase state produces variance under 1%.
- **SC-005**: Zero external dependencies — the skill works with only Claude Code installed.
- **SC-006**: The maturity level correctly differentiates repositories benchmarked in the framework (e.g., CockroachDB at L4, Flask at L2, Express at L2).
- **SC-007**: Level progression is gated — a repository cannot reach Level 3 by excelling at Level 4 criteria while neglecting Level 1-2 fundamentals.

## Clarifications

### Session 2026-05-11

- Q: What format should the primary report output be? → A: Web-based HTML report with dark theme, featuring: project header (name, language badge, repo path, pass rate), level progress bar (L1-L5 with completion percentages), narrative summary with Strengths and Opportunities columns, and per-pillar criteria breakdown with ✓/✗/— indicators showing criterion name (snake_case), score (1/1, 0/1, —/—), and evidence text.
- Q: Should criteria requiring GitHub API access (branch protection, issue labeling, secret scanning) require `gh` CLI? → A: Use `gh` CLI when authenticated, gracefully skip API-dependent criteria when not available (mark as —/— with explanation).
- Q: Should v1 include automated remediation (generating fixes for MISSING criteria)? → A: No. Report only (read-only) for v1. Automated remediation is a follow-on feature after scoring is validated.

## Assumptions

- The skill is invoked inside a project directory containing source code.
- Assessment is static analysis only — no runtime testing, no network calls, no code execution beyond reading files.
- The skill runs within Claude Code and has access to the filesystem via Claude Code's tools.
- Supported languages for v1: Go, Python, TypeScript/JavaScript, Rust, C++, Swift.
- Criteria not applicable to a repository's nature are skipped and do not count against the score.
- Monorepo support uses application-scoped evaluation (per-app scores like "3/4") where applicable.
- The 37-repository benchmark dataset serves as the validation baseline for scoring accuracy.
- Grounding on previous reports is used to tame non-determinism (variance target: under 1%).
- Automated remediation (generating fixes, opening PRs) is explicitly out of scope for v1. The skill is strictly read-only and report-only.
