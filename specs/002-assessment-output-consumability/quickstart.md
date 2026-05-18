# Quickstart: Assessment Output Consumability

## Prerequisites

- Feature branch checked out: `002-assessment-output-consumability`
- Agent-Fit plugin available in current workspace
- A target repository for running `/agentfit`

## Validate Enhanced Outputs

1. Run `/agentfit` in a test repository.
2. Confirm both output files exist:
   - `/tmp/agentfit-report-{project_name}.html`
   - `/tmp/agentfit-report-{project_name}.json`
3. Open HTML and verify:
   - Footer includes assessment date, skill version, criteria totals, git SHA, and duration
   - Pillar collapse/expand works
   - Status filters work (`Show All`, `Found`, `Missing`, `Skipped`)
   - Search filters criteria by name
   - Level filter narrows rows by maturity level
4. Confirm console output includes `Levels:` lines with gate status symbols.

## Validate Baseline Delta Behavior

1. Ensure a sidecar baseline exists at `/tmp/agentfit-report-{project_name}.json`.
2. Make a small repository change that alters at least one criterion.
3. Re-run `/agentfit`.
4. Verify HTML displays:
   - Pass-rate delta (for example `+5%`)
   - Highlighted criteria with changed status
5. Verify JSON includes delta fields and criterion-level change markers.

## Validate No-Baseline Compatibility

1. Remove or rename `/tmp/agentfit-report-{project_name}.json`.
2. Run `/agentfit`.
3. Confirm run succeeds and produces both outputs without delta values.

## Regression Checks

- Existing score calculations and maturity gating still match previous logic.
- Existing pillar ordering and criterion ordering remain unchanged.
- Summary headline quality improves with expanded examples and simplified template language.
