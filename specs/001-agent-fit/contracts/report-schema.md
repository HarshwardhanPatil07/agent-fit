# Report Output Contract: Agent-Fit HTML Report

**Date**: 2026-05-11

## Output Format

The skill generates an HTML file and opens it in the default browser.

## HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>{project_name} — Agent Fit Report</title>
  <style>/* Dark theme CSS inline */</style>
</head>
<body>
  <!-- HEADER -->
  <header>
    <h1>{project_name} <span class="badge">{language}</span></h1>
    <p class="meta">{repo_path}  PASS RATE {pass_rate}%</p>
    <p class="description">{project_description}</p>
  </header>

  <!-- LEVEL PROGRESS BAR -->
  <div class="level-bar">
    <div class="level-segment l1" style="width:{l1_pct}%">
      {l1_pct}% L1
    </div>
    <div class="level-segment l2" style="width:{l2_pct}%">
      {l2_pct}% L2
    </div>
    <!-- ... L3, L4, L5 -->
  </div>

  <!-- SUMMARY -->
  <section class="summary">
    <h2>{summary_headline}</h2>
    <p>{summary_text}</p>

    <div class="columns">
      <div class="strengths">
        <h3>STRENGTHS</h3>
        <!-- 01, 02, 03 with green accent -->
      </div>
      <div class="opportunities">
        <h3>OPPORTUNITIES</h3>
        <!-- 01, 02, 03 with orange accent -->
      </div>
    </div>
  </section>

  <!-- ALL CRITERIA -->
  <section class="criteria">
    <h3>ALL CRITERIA</h3>

    <!-- Per-pillar group -->
    <div class="pillar-group">
      <div class="pillar-header">
        <span>{pillar_name}</span>
        <span>{passed}/{total} ({percentage}%)</span>
      </div>

      <!-- Per-criterion row -->
      <div class="criterion-row">
        <span class="status">{✓|✗|—}</span>
        <span class="name">{criterion_name}</span>
        <span class="score">{1/1|0/1|—/—}</span>
        <span class="evidence">{evidence_text}</span>
      </div>
    </div>
  </section>
</body>
</html>
```

## Visual Design Tokens (from screenshots)

| Token | Value |
|-------|-------|
| Background | Dark (#1a1a2e or similar) |
| Text | Light gray (#e0e0e0) |
| Pass icon (✓) | Green (#4caf50) |
| Fail icon (✗) | Red/orange (#f44336) |
| Skip icon (—) | Gray (#888) |
| Strength accent | Green |
| Opportunity accent | Orange |
| Level bar L1-L3 | Green gradient |
| Level bar L4 | Orange/yellow |
| Level bar L5 | Gray (unfilled) |
| Language badge | Outlined pill shape |
| Font | Monospace for criteria, sans-serif for headers |
| Section headers | ALL CAPS, small, tracking-wide |

## Criterion Row Format

Each criterion displays as a single row:

```
{status_icon}  {criterion_name}  {score}  {evidence_text}
```

- `status_icon`: ✓ (green), ✗ (red), — (gray)
- `criterion_name`: snake_case, monospace font
- `score`: "1/1" (pass), "0/1" (fail), "—/—" (skipped)
- `evidence_text`: 1-2 sentences describing what was found or what's missing

## Pillar Header Format

```
{Pillar Name}                    {passed}/{total} ({percentage}%)
```

Right-aligned score with pillar name left-aligned. Separator line below.

## Sorting

- Pillars: Fixed order (1-9 as defined in spec)
- Criteria within pillar: Alphabetical by snake_case name
