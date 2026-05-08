# Pillar 9: Product and Experimentation

Scan for 3 signals that measure product feedback loops. This is the most commonly failed pillar.

---

## Signal: `product_analytics`

**Purpose:** User behavior tracking enables data-driven product decisions.

**Detection:**

1. Search dependency manifests:
   - **TypeScript/JavaScript:** `package.json` — `mixpanel`, `@mixpanel/browser`, `amplitude-js`, `@amplitude/analytics-browser`, `posthog-js`, `posthog-node`, `@segment/analytics-next`, `analytics-node`, `@heap/heap-node`, `@googleanalytics/data`, `@google-analytics/data`, `react-ga4`, `gtag`
   - **Python:** `mixpanel`, `amplitude-analytics`, `posthog`, `segment-analytics-python`, `google-analytics-data`
   - **Go:** `github.com/dukex/mixpanel`, `github.com/amplitude/analytics-go`, `github.com/posthog/posthog-go`
   - **Ruby:** `mixpanel-ruby`, `amplitude-api`, `posthog-ruby`
2. Search source code for analytics patterns:
   - `grep -rl "analytics\.\|mixpanel\.\|amplitude\.\|posthog\.\|gtag\.\|dataLayer" --include="*.ts" --include="*.js" --include="*.tsx" --include="*.jsx" --include="*.py" . | head -5`
3. Search for analytics configs:
   - Google Analytics: `GA_TRACKING_ID`, `NEXT_PUBLIC_GA_ID` in env templates
   - Segment: `analytics.js` config
   - PostHog: `POSTHOG_API_KEY`, `NEXT_PUBLIC_POSTHOG_KEY` in env templates

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `error_to_insight_pipeline`

**Purpose:** Errors automatically create trackable issues, closing the feedback loop.

**Detection:**

1. Search for error-to-issue integrations:
   - Sentry + GitHub integration: search for Sentry configs with GitHub project references
   - CI workflows that create issues from errors
   - Error monitoring with auto-issue creation: search for `createIssue`, `create_issue` patterns in error handlers
2. Search for custom error pipelines:
   - Webhook configs that route errors to issue trackers
   - Bot configurations that convert alerts to issues
3. Search dependency manifests:
   - `@sentry/github` integration packages
   - Error-to-Slack-to-issue pipeline tools

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `experiment_infrastructure`

**Purpose:** A/B testing frameworks enable measured, evidence-based feature decisions.

**Detection:**

1. Search dependency manifests:
   - **TypeScript/JavaScript:** `package.json` — `@statsig/js-client`, `@statsig/react`, `@growthbook/growthbook`, `@growthbook/growthbook-react`, `@optimizely/optimizely-sdk`, `@optimizely/react-sdk`, `@split.io/splitio`, `@split.io/splitio-react`, `@absmartly/javascript-sdk`
   - **Python:** `statsig`, `growthbook`, `optimizely-sdk`, `split-python-sdk`
   - **Go:** `github.com/statsig-io/go-sdk`, `github.com/growthbook/growthbook-golang`
2. Search source code for experiment patterns:
   - `grep -rl "experiment\|A/B\|ab_test\|abTest\|variant\|checkGate\|getExperiment\|isInExperiment\|useExperiment\|useFeatureGate" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" . | head -5`
3. Search for experiment configs:
   - Feature flag configs with metrics/analytics integration
   - Experiment definition files

**Output fields:** `found`, `tools`, `config_files`, `evidence`
