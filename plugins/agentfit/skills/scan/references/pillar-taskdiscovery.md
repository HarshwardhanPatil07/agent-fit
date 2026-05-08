# Pillar 8: Task Discovery

Scan for 4 signals that measure how well agents can find and scope work.

---

## Signal: `issue_templates`

**Purpose:** Structured issue templates provide consistent context for agents.

**Detection:**

1. Search for issue template files:
   - `.github/ISSUE_TEMPLATE/` directory
   - `.github/ISSUE_TEMPLATE/bug_report.md`, `bug_report.yml`
   - `.github/ISSUE_TEMPLATE/feature_request.md`, `feature_request.yml`
   - `.github/ISSUE_TEMPLATE/config.yml` (template chooser config)
   - `.github/ISSUE_TEMPLATE.md` (single template, legacy format)
   - `.gitlab/issue_templates/` (GitLab)
2. If found, list template types:
   - Parse filenames to identify types: bug report, feature request, question, etc.

**Output fields:** `found`, `tools`, `config_files`, `template_types` (list like `["bug_report", "feature_request"]`), `evidence`

---

## Signal: `issue_labeling_system`

**Purpose:** A comprehensive label taxonomy enables filtering and prioritization.

**Detection (remote only — requires GitHub API):**

1. Fetch labels: `gh api repos/{owner}/{repo}/labels --paginate -q '.[].name'`
   - If 403/404, record `api_accessible: false`
2. Analyze label quality:
   - Count total labels
   - Check for priority labels: `P0`, `P1`, `P2`, `P3`, `priority:*`, `critical`, `high`, `medium`, `low`
   - Check for type labels: `bug`, `enhancement`, `feature`, `documentation`, `question`
   - Check for area/component labels: `area:*`, `component:*`, `pkg:*`
   - Check for status labels: `in-progress`, `needs-review`, `blocked`, `wontfix`
3. For local scans, set `found: null`

**Output fields:** `found` (null for local or API-inaccessible), `tools`, `config_files`, `api_accessible`, `label_count`, `labels` (list of label names, capped at 50), `error`, `evidence`

---

## Signal: `backlog_health`

**Purpose:** Well-described, labeled issues enable agents to autonomously pick up work.

**Detection (remote only — requires GitHub API):**

1. Fetch recent issues: `gh api "repos/{owner}/{repo}/issues?per_page=20&state=all" -q '.[] | {title: .title, body: (.body // "" | length), labels: [.labels[].name], state: .state}'`
   - If 403/404, record `api_accessible: false`
2. Analyze quality:
   - `titled_pct`: percentage with titles longer than 10 characters
   - `labeled_pct`: percentage with at least one label
   - `avg_body_length`: average body length in characters
3. For local scans, set `found: null`

**Output fields:** `found` (null for local or API-inaccessible), `tools`, `config_files`, `api_accessible`, `sample_size`, `titled_pct`, `labeled_pct`, `avg_body_length`, `error`, `evidence`

---

## Signal: `pr_templates`

**Purpose:** PR templates ensure structured descriptions and testing checklists.

**Detection:**

1. Search for PR template files:
   - `.github/pull_request_template.md`
   - `.github/PULL_REQUEST_TEMPLATE.md`
   - `.github/PULL_REQUEST_TEMPLATE/` directory (multiple templates)
   - `docs/pull_request_template.md`
   - `PULL_REQUEST_TEMPLATE.md` (root level)
   - `.gitlab/merge_request_templates/` (GitLab)
2. If found, check for key sections:
   - Description, testing, checklist, related issues

**Output fields:** `found`, `tools`, `config_files`, `evidence`
