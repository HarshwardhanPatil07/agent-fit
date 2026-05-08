# Pillar 6: Debugging and Observability

Scan for 11 signals that measure runtime visibility and diagnostic capability.

---

## Signal: `structured_logging`

**Purpose:** Structured logs (JSON, key-value) are machine-parseable, enabling agents to analyze runtime behavior.

**Detection:**

1. Search dependency manifests for logging libraries:
   - **Go:** `go.mod` — `go.uber.org/zap`, `github.com/sirupsen/logrus`, `github.com/rs/zerolog`, `log/slog` (stdlib), `github.com/charmbracelet/log`
   - **Python:** `pyproject.toml` / `requirements.txt` — `structlog`, `python-json-logger`, `loguru`; also check for stdlib `import logging` with JSON formatter
   - **TypeScript/JavaScript:** `package.json` — `winston`, `pino`, `bunyan`, `log4js`, `loglevel`, `tslog`
   - **Rust:** `Cargo.toml` — `tracing`, `log`, `env_logger`, `slog`, `flexi_logger`
   - **Ruby:** `Gemfile` — `lograge`, `semantic_logger`, `ougai`
   - **Java:** `pom.xml` / `build.gradle` — `logback`, `log4j2`, `slf4j`
   - **C++:** check for `spdlog`, ETW TraceLogging, `glog`
2. Search source code for structured logging patterns:
   - `grep -rl "zap\.\|zerolog\.\|structlog\.\|winston\.\|pino\.\|tracing::" --include="*.go" --include="*.py" --include="*.ts" --include="*.js" --include="*.rs" . | head -5`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `distributed_tracing`

**Purpose:** Request tracing with trace/span IDs enables end-to-end debugging.

**Detection:**

1. Search dependency manifests:
   - **Go:** `go.mod` — `go.opentelemetry.io/otel`, `github.com/opentracing/opentracing-go`, `github.com/DataDog/dd-trace-go`
   - **Python:** `opentelemetry-api`, `opentelemetry-sdk`, `ddtrace`, `jaeger-client`
   - **TypeScript/JavaScript:** `@opentelemetry/api`, `@opentelemetry/sdk-node`, `dd-trace`, `@sentry/tracing`
   - **Rust:** `opentelemetry`, `tracing-opentelemetry`
   - **Java:** `opentelemetry-api`, `spring-cloud-sleuth`, `brave`
2. Search for tracing configs:
   - `otel-collector-config.yaml`
   - OpenTelemetry environment variables in `.env.example` or docs
3. Search source code:
   - `grep -rl "X-Request-ID\|trace_id\|TraceID\|span_id\|SpanID\|opentelemetry\|otel" --include="*.go" --include="*.py" --include="*.ts" --include="*.js" --include="*.rs" . | head -5`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `metrics_collection`

**Purpose:** Application metrics enable data-driven decision making.

**Detection:**

1. Search dependency manifests:
   - **Go:** `go.mod` — `github.com/prometheus/client_golang`, `github.com/DataDog/datadog-go`, `go.opentelemetry.io/otel/metric`
   - **Python:** `prometheus-client`, `datadog`, `opentelemetry-api`
   - **TypeScript/JavaScript:** `prom-client`, `hot-shots` (StatsD), `@opentelemetry/api`
   - **Rust:** `prometheus`, `metrics`, `opentelemetry`
2. Search for metrics configs:
   - `prometheus.yml`, `prometheus.yaml`
   - `datadog.yaml`
   - Grafana dashboard JSON files
3. Search source code for metric patterns:
   - `grep -rl "prometheus\.\|metrics\.\|statsd\.\|counter\.\|histogram\.\|gauge\." --include="*.go" --include="*.py" --include="*.ts" --include="*.js" . | head -5`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `alerting_configured`

**Purpose:** Alerts fire on anomalies, enabling proactive response.

**Detection:**

1. Search for alerting configs:
   - Prometheus alert rules: `*.rules.yml`, `alerts.yml`, `alerting_rules/`
   - PagerDuty configs: `pagerduty` references in configs
   - OpsGenie configs
   - Grafana alert definitions in dashboard JSON
2. Search CI/deployment configs for alerting:
   - Alert channel configs in deployment manifests
   - Slack/email notification configs for monitoring

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `deployment_observability`

**Purpose:** Dashboards track deployment health in real time.

**Detection:**

1. Search for dashboard configs:
   - Grafana dashboard JSON files: `dashboards/`, `grafana/`, `monitoring/dashboards/`
   - `docker-compose.yml` with Grafana service
   - Dashboard links in documentation
2. Search for deployment monitoring:
   - Deployment notification configs (Slack, webhook)
   - Deploy tracking in CI (annotations, markers)
3. Search docs:
   - `README.md`, `docs/` — grep for "grafana", "dashboard", "monitoring", "observability"

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `error_tracking`

**Purpose:** Error tracking with release context enables targeted debugging.

**Detection:**

1. Search dependency manifests:
   - **All languages:** `sentry-sdk`, `@sentry/node`, `@sentry/react`, `@sentry/browser`, `sentry-go`, `sentry`
   - `bugsnag`, `@bugsnag/js`, `bugsnag-go`
   - `rollbar`, `@rollbar/react`
   - `airbrake`, `honeybadger`
   - `raygun4js`, `raygun4node`
2. Search for error tracking configs:
   - `.sentryclirc`, `sentry.properties`
   - Sentry DSN in environment configs
3. Search source code:
   - `grep -rl "Sentry\.\|sentry_sdk\.\|Bugsnag\.\|Rollbar\." --include="*.go" --include="*.py" --include="*.ts" --include="*.js" . | head -5`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `health_checks`

**Purpose:** Health/readiness endpoints verify service availability.

**Detection:**

1. Search source code for health endpoints:
   - `grep -rn '"/health"\|"/healthz"\|"/ready"\|"/readyz"\|"/livez"\|"/ping"\|"/status"' --include="*.go" --include="*.py" --include="*.ts" --include="*.js" --include="*.rs" --include="*.rb" . | head -10`
2. Search for Kubernetes probes:
   - `grep -rn "livenessProbe\|readinessProbe\|startupProbe" --include="*.yaml" --include="*.yml" . | head -5`
3. Search for health check libraries:
   - `package.json`: `@godaddy/terminus`, `lightship`
   - Go: custom health handler patterns

**Output fields:** `found`, `tools`, `config_files`, `endpoints` (list of detected health endpoints), `evidence`

---

## Signal: `profiling`

**Purpose:** CPU/memory profiling infrastructure enables performance diagnosis.

**Detection:**

1. Search for profiling tools:
   - **Go:** `net/http/pprof` import in source code, or `pkg/server/profiler/`
   - **Python:** `py-spy`, `pyinstrument`, `cProfile`, `memory_profiler` in dependencies
   - **Rust:** `pprof-rs`, `tracy-client`, `flamegraph` in Cargo.toml
   - **TypeScript/JavaScript:** `clinic`, `0x`, `v8-profiler-next` in package.json
   - **C++:** `tracy`, `gperftools`, Performance.md
2. Search source code:
   - `grep -rl "pprof\|profiler\|pyinstrument\|cProfile" --include="*.go" --include="*.py" --include="*.ts" --include="*.rs" . | head -5`
3. Search for profiling documentation:
   - `PERFORMANCE.md`, `docs/profiling/`, `docs/performance/`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `code_quality_metrics`

**Purpose:** Code quality tracking over time via CI integration.

**Detection:**

1. Search for quality tool configs:
   - `codecov.yml`, `.codecov.yml`
   - `.codeclimate.yml`
   - `sonar-project.properties`, `sonar-project.json`
   - `.deepsource.toml`
2. Search CI workflows for quality integrations:
   - `codecov/codecov-action`
   - `codeclimate/test-reporter`
   - `SonarSource/sonarcloud-github-action`
   - `deepsource`
   - `codacy` actions
   - Coverage upload steps

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `circuit_breakers`

**Purpose:** Resilience patterns prevent cascading failures from external dependencies.

**Detection:**

1. Search dependency manifests:
   - **Go:** `github.com/hashicorp/go-retryablehttp`, `github.com/sony/gobreaker`, `github.com/afex/hystrix-go`, `github.com/cenkalti/backoff`
   - **Python:** `pybreaker`, `tenacity`, `circuitbreaker`
   - **TypeScript/JavaScript:** `cockatiel`, `opossum`, `brakes`, `p-retry`, `retry-axios`
   - **Rust:** `tower` (middleware with retry/circuit breaker), `backoff`
   - **Java:** `resilience4j`, `hystrix`, `spring-retry`
2. Search source code:
   - `grep -rl "circuit.breaker\|CircuitBreaker\|retryablehttp\|gobreaker\|Retry\|retry_policy" --include="*.go" --include="*.py" --include="*.ts" --include="*.js" . | head -5`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `runbooks_documented`

**Purpose:** Operational runbooks guide incident response.

**Detection:**

1. Search for runbook files:
   - `runbooks/`, `runbook/`, `docs/runbooks/`, `docs/runbook/`
   - `RUNBOOK.md`, `INCIDENT_RESPONSE.md`, `ONCALL.md`
   - `docs/operations/`, `docs/ops/`, `docs/incidents/`
2. Search documentation for operational procedures:
   - `README.md` — grep for "runbook", "incident", "on-call", "oncall", "playbook"

**Output fields:** `found`, `tools`, `config_files`, `evidence`
