# Pillar 3: Testing

Scan for 8 signals that indicate test infrastructure quality and coverage.

---

## Signal: `unit_tests_exist`

**Purpose:** Unit tests are the primary mechanism by which agents validate their changes.

**Detection:**

1. Search for test files by language pattern:
   - Go: `find . -name "*_test.go" -not -path "*vendor*" | head -10`
   - Python: `find . -name "test_*.py" -o -name "*_test.py" | head -10`
   - TypeScript/JavaScript: `find . \( -name "*.test.ts" -o -name "*.test.tsx" -o -name "*.test.js" -o -name "*.test.jsx" -o -name "*.spec.ts" -o -name "*.spec.tsx" -o -name "*.spec.js" -o -name "*.spec.jsx" \) -not -path "*/node_modules/*" | head -10`
   - Rust: grep for `#[test]` or `#[cfg(test)]` in `.rs` files: `grep -rl "#\[test\]\|#\[cfg(test)\]" --include="*.rs" . | head -10`
   - Ruby: `find . -name "*_test.rb" -o -name "*_spec.rb" | head -10`
   - Java: `find . -name "*Test.java" -o -name "*Tests.java" | head -10`
   - Swift: `find . -name "*Tests.swift" -o -name "*Test.swift" | head -10`
   - C++: `find . -name "*_test.cpp" -o -name "*_test.cc" -o -name "test_*.cpp" | head -10`
2. Count total test files: `find . -name "<pattern>" | wc -l`

**Output fields:** `found`, `tools`, `config_files`, `test_file_count`, `sample_paths` (up to 5 example paths), `evidence`

---

## Signal: `unit_tests_runnable`

**Purpose:** Tests must be executable via a documented command.

**Detection:**

1. Search for test commands in documentation:
   - `README.md`, `CONTRIBUTING.md`, `CLAUDE.md`, `AGENTS.md` — grep for `test` in code blocks
   - `Makefile` — check for `test` target
   - `package.json` — check for `test` script
   - `pyproject.toml` — check for `[tool.pytest.ini_options]` or `[tool.hatch.envs.*.scripts]`
   - `tox.ini` — check for test environments
2. Search for test framework configs:
   - `jest.config.js`, `jest.config.ts`, `jest.config.mjs`, `jest.config.cjs`, `jest.config.json`
   - `vitest.config.ts`, `vitest.config.js`, `vitest.config.mts`
   - `pytest.ini`, `pyproject.toml` with `[tool.pytest]`, `setup.cfg` with `[tool:pytest]`, `conftest.py`
   - `karma.conf.js`
   - `.rspec`
   - `Cargo.toml` (cargo test is built-in for Rust)
   - Go: `go test` is built-in

**Output fields:** `found`, `tools`, `config_files`, `test_command` (the documented test command if found), `evidence`

---

## Signal: `integration_tests_exist`

**Purpose:** Integration/E2E tests validate system-level behavior.

**Detection:**

1. Search for integration test directories:
   - `test/integration/`, `tests/integration/`, `integration_tests/`, `e2e/`, `test/e2e/`, `tests/e2e/`
   - `pkg/acceptance/`, `acceptance/`, `test/acceptance/`
   - `cypress/`, `playwright/`, `test/playwright/`
2. Search for E2E framework configs:
   - `cypress.config.js`, `cypress.config.ts`, `cypress.json`
   - `playwright.config.ts`, `playwright.config.js`
   - `nightwatch.conf.js`
   - `wdio.conf.js`, `wdio.conf.ts`
3. Search for integration test files:
   - Files matching `*_integration_test.go`, `*_e2e_test.go`
   - Files matching `*_integration_test.py`, `test_*_integration.py`
   - `compiletest` directories (Rust)
4. Search dependency manifests:
   - `package.json`: `cypress`, `playwright`, `@playwright/test`, `puppeteer`, `selenium-webdriver`
   - `pyproject.toml`: `selenium`, `playwright`

**Output fields:** `found`, `tools`, `config_files`, `test_file_count`, `sample_paths` (up to 5), `evidence`

---

## Signal: `test_coverage_thresholds`

**Purpose:** Coverage thresholds prevent test coverage from decreasing.

**Detection:**

1. Search for coverage configs:
   - `codecov.yml`, `.codecov.yml` — check for `target` or `threshold` settings
   - `jest.config.*` — check for `coverageThreshold`
   - `vitest.config.*` — check for `coverage.thresholds`
   - `pyproject.toml` — check for `[tool.coverage.report]` with `fail_under`
   - `.coveragerc` — check for `fail_under`
   - `setup.cfg` — check for `[coverage:report]` with `fail_under`
   - `tox.ini` — check for coverage flags
   - `.nycrc`, `.nycrc.json` — check for `check-coverage`, `branches`, `lines`, `functions`
   - `sonar-project.properties` — check for coverage thresholds
2. Search CI workflows:
   - Steps running coverage with thresholds: `--cov-fail-under`, `--coverage`, `smokeshow`
   - Coverage upload actions: `codecov/codecov-action`, `coverallsapp`

**Output fields:** `found`, `tools`, `config_files`, `thresholds` (e.g., `{"lines": 80, "branches": 70}`), `evidence`

---

## Signal: `test_naming_conventions`

**Purpose:** Consistent test naming helps agents find and create tests.

**Detection:**

1. Check if test files follow language conventions:
   - Go: all tests in `*_test.go` files, functions start with `Test`
   - Python: files match `test_*.py` or `*_test.py`, functions start with `test_`
   - TypeScript/JavaScript: files match `*.test.ts` or `*.spec.ts`
   - Rust: test modules use `#[cfg(test)]` and functions use `#[test]`
   - Ruby: files match `*_spec.rb` or `*_test.rb`
2. Sample 5-10 test files and verify naming consistency

**Output fields:** `found`, `tools`, `config_files`, `patterns` (list of naming patterns found, e.g., `["*_test.go", "Test*"]`), `evidence`

---

## Signal: `test_isolation`

**Purpose:** Isolated tests run independently without shared state, enabling parallel execution.

**Detection:**

1. Search for parallel test execution:
   - Go: grep for `t.Parallel()` in test files, or `-race` flag in test commands
   - Python: `pytest-xdist` in dependencies, `-n auto` or `-n <N>` in pytest config
   - Jest: `--workers`, `--maxWorkers` in jest config
   - Vitest: `--pool forks`, `--pool threads` in config
   - Rust: `cargo nextest` in dependencies or CI
2. Search CI for parallel test execution:
   - Steps with `--parallel`, `-p`, `-j`, `--workers`
   - Matrix strategies in CI workflows that shard tests
3. Search for test isolation patterns:
   - `beforeEach`/`afterEach` cleanup in JS test files
   - `setUp`/`tearDown` in Python test files
   - `t.TempDir()` in Go test files

**Output fields:** `found`, `tools`, `config_files`, `parallel_execution` (true if parallel test execution is configured), `evidence`

---

## Signal: `flaky_test_detection`

**Purpose:** Flaky tests waste agent time with non-deterministic failures.

**Detection:**

1. Search for retry/rerun configs:
   - `package.json`: `jest-circus` with retry, `jest-retry`
   - `pyproject.toml` / `requirements.txt`: `pytest-rerunfailures`, `flaky`
   - Go: `--stress` flag, `--count` flag in test commands
   - CI: `retry-on-failure` patterns
2. Search CI for flaky test handling:
   - Steps with `--retry`, `--rerun`, `--flaky`
   - Quarantine/skip patterns for known flaky tests
   - Flaky test dashboards or tracking
3. Search for test retry configs:
   - Jest `retries` in config
   - Cypress `retries` in config
   - Playwright `retries` in config

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `test_performance_tracking`

**Purpose:** Tracking test execution times identifies slow tests and prevents regression.

**Detection:**

1. Search for test timing tools:
   - pytest `--durations` flag in CI or config
   - Jest `--verbose` with timing
   - Go: `gotestsum` with `--junitfile` (captures timing)
   - CI: test analytics or timing reports
2. Search for test performance configs:
   - `gotestsum` in Makefile or CI
   - `junit-report` generation in CI
   - Test timing thresholds in CI configs
3. Search for slow test detection:
   - pytest `--timeout` in config
   - Jest `testTimeout` in config

**Output fields:** `found`, `tools`, `config_files`, `evidence`
