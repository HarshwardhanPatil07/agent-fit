# Data Model: Agent-Fit v2 Architecture & Future

**Date**: 2026-05-13
**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

## Entities

### ProjectType

Represents the detected classification of the target codebase.

| Field | Type | Description |
|-------|------|-------------|
| types | set of enum | One or more of: `library`, `cli`, `web_app`, `api_service`, `monorepo` |
| primary_language | string | Primary language detected (from v1 Step 1) |
| detection_evidence | map[type → string] | Per-type evidence explaining why detected (e.g., "Found `bin` field in package.json") |

**Rules**:
- Multiple types can be detected simultaneously
- Detection is deterministic: same codebase always produces same types
- At least one type MUST be detected (fallback: `web_app` as most permissive)

### ApplicabilityMatrix

Centralized mapping from project types to skipped criteria.

| Field | Type | Description |
|-------|------|-------------|
| matrix | map[type → set of criterion_name] | Per-type set of criteria to skip |

**Static matrix definition**:

| Criterion | Library | CLI | Web App | API Service | Monorepo |
|-----------|---------|-----|---------|-------------|----------|
| deployment_frequency | SKIP | — | — | — | — |
| progressive_rollout | SKIP | — | — | — | — |
| rollback_automation | SKIP | — | — | — | — |
| health_checks | SKIP | SKIP | — | — | — |
| product_analytics | SKIP | SKIP | — | SKIP | — |
| local_services_setup | SKIP | — | — | — | — |
| dast_scanning | SKIP | SKIP | — | — | — |
| distributed_tracing | — | SKIP | — | — | — |
| deployment_observability | — | SKIP | — | — | — |
| heavy_dependency_detection | — | — | — | SKIP | — |
| n_plus_one_detection | SKIP | SKIP | — | — | — |

**Resolution rule**: When multiple types detected, skip only if ALL types would skip (union/intersection of non-skip sets).

**Override rule**: `.agentfit.yml` `disabled_criteria` overrides matrix (force-skip). Custom criteria in `.agentfit.yml` are always evaluated regardless of matrix.

### CustomCriteria

User-defined criteria from `.agentfit.yml`.

| Field | Type | Description |
|-------|------|-------------|
| name | string (snake_case) | Unique criterion identifier |
| pillar | string | Must match one of the 9 pillar names |
| level | enum (L1-L5) | Maturity level classification |
| check | string | What the skill should look for |
| found_when | string | Condition that satisfies the criterion |
| impact_tier | enum (high, medium, low) | Optional; defaults to `medium` |

**Validation rules**:
- `name` must be unique across default + custom criteria
- `pillar` must match an existing pillar name exactly
- `level` must be L1-L5
- If validation fails, warn and skip the invalid criterion (not the entire file)

### ImpactTier

Classification of criteria importance for weighted scoring.

| Field | Type | Description |
|-------|------|-------------|
| tier | enum | `high` (3x), `medium` (2x), `low` (1x) |
| multiplier | integer | Weight multiplier for scoring |

**Default tier assignments**:

| Tier | Criteria |
|------|----------|
| High (3x) | type_check, lint_config, unit_tests_exist, agents_md, readme, deps_pinned, secrets_management, gitignore_comprehensive, pre_commit_hooks, single_command_setup |
| Low (1x) | duplicate_code_detection, tech_debt_tracking, code_modularization, naming_consistency, build_performance_tracking, dead_feature_flag_detection, heavy_dependency_detection, privacy_compliance |
| Medium (2x) | All remaining criteria |

### ReportMetadata

Metadata included in HTML footer and JSON output.

| Field | Type | Description |
|-------|------|-------------|
| schema_version | string (semver) | From plugin.json `version` |
| assessment_date | ISO 8601 datetime | When the assessment was run |
| git_sha | string | HEAD commit SHA of the assessed repo |
| total_criteria | integer | Total criteria evaluated (excluding skipped) |
| skipped_criteria | integer | Number of criteria skipped |
| project_types | array of string | Detected project types |
| weighted_score | float | Weighted pass rate (0-100) |
| pass_rate | float | Unweighted pass rate (0-100) |

## JSON Output Schema (extended)

The existing JSON output gains these top-level fields:

```json
{
  "schema_version": "2.0.0",
  "metadata": {
    "assessment_date": "2026-05-13T14:30:00Z",
    "git_sha": "abc123def",
    "skill_version": "2.0.0",
    "project_types": ["cli", "monorepo"],
    "total_criteria": 68,
    "skipped_criteria": 7,
    "custom_criteria_count": 2
  },
  "scores": {
    "pass_rate": 67.0,
    "weighted_score": 72.0,
    "maturity_level": 3
  },
  "pillars": { "...existing structure..." },
  "criteria": { "...existing structure with impact_tier field added..." }
}
```
