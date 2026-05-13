# Contract: `.agentfit.yml` Configuration Schema

**Date**: 2026-05-13
**Spec**: [spec.md](../spec.md)

## Overview

The `.agentfit.yml` file is an optional configuration file placed at the repository root. It allows teams to customize the agent-fit assessment by adding project-specific criteria and disabling irrelevant default criteria.

## Schema

```yaml
# .agentfit.yml — Agent-Fit custom configuration
# All fields are optional. An empty or missing file uses defaults.

custom_criteria:
  - name: "criterion_snake_case_name"   # Required. Must be unique across all criteria.
    pillar: "Documentation"              # Required. Must match one of the 9 pillar names exactly:
                                         #   Style & Validation, Build System, Testing,
                                         #   Documentation, Development Environment,
                                         #   Debugging & Observability, Security & Governance,
                                         #   Task Discovery, Product & Experimentation
    level: "L2"                          # Required. One of: L1, L2, L3, L4, L5
    check: "docs/api/ directory exists"  # Required. What the skill should look for.
    found_when: "docs/api/ contains at least one file"  # Required. Condition for FOUND status.
    impact_tier: "medium"                # Optional. One of: high, medium, low. Default: medium.

disabled_criteria:
  - "product_analytics_instrumentation"  # Criterion names to fully remove from assessment.
  - "dast_scanning"                      # Must match default criterion names exactly.
```

## Validation Rules

1. File MUST be valid YAML. If parsing fails, warn and proceed with defaults.
2. `custom_criteria[].name` MUST be snake_case and unique across default + custom criteria.
3. `custom_criteria[].pillar` MUST match one of the 9 pillar names exactly (case-sensitive).
4. `custom_criteria[].level` MUST be one of: L1, L2, L3, L4, L5.
5. `disabled_criteria[]` entries MUST match default criterion names. Unknown names are warned and ignored.
6. Per-criterion validation: if a single custom criterion is invalid, skip it with a warning but process the rest.

## Precedence

`.agentfit.yml` always overrides the project-type applicability matrix:
- A criterion in `disabled_criteria` is removed even if the project type would evaluate it.
- A custom criterion is always evaluated regardless of project type.
- If a default criterion is both in the applicability matrix skip list AND in `disabled_criteria`, it is removed (not shown as skipped).

## Example

```yaml
custom_criteria:
  - name: "internal_api_docs"
    pillar: "Documentation"
    level: "L2"
    check: "docs/api/ directory with endpoint documentation"
    found_when: "docs/api/ contains at least one .md or .yaml file"
  - name: "staging_environment"
    pillar: "Development Environment"
    level: "L3"
    check: "Staging environment configuration exists"
    found_when: "docker-compose.staging.yml or staging/ directory found"

disabled_criteria:
  - "product_analytics_instrumentation"
  - "experiment_infrastructure"
```
