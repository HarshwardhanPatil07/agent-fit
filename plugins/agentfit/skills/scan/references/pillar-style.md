# Pillar 1: Style and Validation

Scan for 13 signals that indicate automated code quality enforcement.

---

## Signal: `formatter`

**Purpose:** Automated code formatting prevents style drift and reduces review friction for agents.

**Detection:**

1. Search for config files:
   - `.prettierrc`, `.prettierrc.json`, `.prettierrc.yml`, `.prettierrc.yaml`, `.prettierrc.js`, `.prettierrc.cjs`, `.prettierrc.mjs`, `.prettierrc.toml`, `prettier.config.js`, `prettier.config.cjs`, `prettier.config.mjs`
   - `.editorconfig` (partial — formatting hints only)
   - `biome.json`, `biome.jsonc`
   - `rustfmt.toml`, `.rustfmt.toml`
   - `.clang-format`
   - `pyproject.toml` — check for `[tool.black]` or `[tool.ruff.format]` sections
   - `setup.cfg` — check for `[tool:black]` or `[isort]`
   - `.style.yapf`, `.yapfignore`
   - `gofmt` / `gofumpt` — check Makefile or CI for `gofmt` / `gofumpt` commands
   - `crlfmt` — check Makefile or CI
2. Search dependency manifests:
   - `package.json` devDependencies: `prettier`, `@biomejs/biome`
   - `pyproject.toml` dependencies: `black`, `ruff`, `yapf`, `autopep8`, `isort`
   - `Gemfile`: `rubocop` (combined formatter+linter)
3. Search CI workflows (`.github/workflows/*.yml`):
   - Steps containing `format`, `fmt`, `prettier`, `black`, `ruff format`, `gofmt`, `gofumpt`, `rustfmt`, `clang-format`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `linter`

**Purpose:** Linters catch bugs, enforce conventions, and prevent anti-patterns.

**Detection:**

1. Search for config files:
   - `eslint.config.js`, `eslint.config.mjs`, `eslint.config.cjs`, `.eslintrc`, `.eslintrc.js`, `.eslintrc.json`, `.eslintrc.yml`, `.eslintrc.yaml`, `.eslintrc.cjs`
   - `biome.json`, `biome.jsonc` (linter section)
   - `.golangci.yml`, `.golangci.yaml`, `.golangci.json`, `.golangci.toml`
   - `pyproject.toml` — check for `[tool.ruff.lint]`, `[tool.pylint]`, `[tool.flake8]`
   - `.flake8`, `setup.cfg` with `[flake8]`
   - `.pylintrc`, `pylintrc`
   - `clippy.toml`, `.clippy.toml`
   - `.rubocop.yml`
   - `.swiftlint.yml`
   - `cppcheck.cfg`, `.clang-tidy`
2. Search dependency manifests:
   - `package.json`: `eslint`, `@biomejs/biome`, `oxlint`
   - `pyproject.toml`: `ruff`, `pylint`, `flake8`, `pyflakes`, `pycodestyle`
   - `Cargo.toml`: clippy is built-in (check CI for `cargo clippy`)
3. Search CI workflows:
   - Steps containing `lint`, `eslint`, `clippy`, `golangci-lint`, `ruff check`, `pylint`, `flake8`, `rubocop`, `swiftlint`

**Output fields:** `found`, `tools`, `config_files`, `rules_configured` (true if custom rules beyond defaults), `evidence`

---

## Signal: `type_checker`

**Purpose:** Static type checking catches type errors before runtime.

**Detection:**

1. Search for config files:
   - `tsconfig.json`, `tsconfig.*.json`, `jsconfig.json`
   - `mypy.ini`, `.mypy.ini`, `pyproject.toml` with `[tool.mypy]` or `[mypy]`
   - `pyrightconfig.json`, `pyproject.toml` with `[tool.pyright]`
   - `pyproject.toml` with `[tool.pytype]`
2. Language-inherent type checking:
   - Go: compiler enforces types (always pass if Go detected)
   - Rust: compiler enforces types (always pass if Rust detected)
   - C/C++: compiler enforces types (always pass if C/C++ detected)
   - Java/Kotlin: compiler enforces types (always pass)
   - Swift: compiler enforces types (always pass)
3. Search CI workflows:
   - Steps containing `tsc`, `mypy`, `pyright`, `pytype`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `strict_typing`

**Purpose:** Strict mode catches more errors — nullable types, implicit any, etc.

**Detection:**

1. For TypeScript — read `tsconfig.json`:
   - Check `compilerOptions.strict` is `true`
   - Or check individual flags: `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitAny`, `noImplicitThis`
2. For Python mypy — read `mypy.ini` or `pyproject.toml [tool.mypy]`:
   - Check for `strict = true` or `strict = True`
   - Or individual flags: `disallow_untyped_defs`, `disallow_any_generics`, `warn_return_any`
3. For Python pyright — read `pyrightconfig.json` or `pyproject.toml [tool.pyright]`:
   - Check `typeCheckingMode` is `"strict"` or `"all"`
4. For Go: strict by default (always pass if Go detected)
5. For Rust: strict by default (always pass if Rust detected). Check for `#![deny(warnings)]` or Clippy pedantic in CI
6. For C++: check for `-Wall -Werror` or `/W4 /WX` in CMakeLists.txt or Makefile

**Output fields:** `found`, `tools`, `config_files`, `strict_settings` (key-value of strict flags found), `evidence`

---

## Signal: `pre_commit_hooks`

**Purpose:** Git hooks provide instant feedback before commits reach CI.

**Detection:**

1. Search for config files:
   - `.pre-commit-config.yaml`
   - `.husky/` directory (check for `pre-commit` file inside)
   - `lefthook.yml`, `.lefthook.yml`
   - `.lintstagedrc`, `.lintstagedrc.json`, `.lintstagedrc.yml`, `.lintstagedrc.js`, `.lintstagedrc.cjs`, `.lintstagedrc.mjs`
   - `lint-staged.config.js`, `lint-staged.config.mjs`, `lint-staged.config.cjs`
   - `package.json` — check for `lint-staged` key
   - `.git/hooks/pre-commit` (if accessible)
   - `scripts/pre-commit.sh`, `scripts/pre-commit`
2. Search dependency manifests:
   - `package.json`: `husky`, `lint-staged`, `lefthook`, `simple-git-hooks`
   - `pyproject.toml`: `pre-commit`

**Output fields:** `found`, `tools`, `config_files`, `hooks_list` (names of hooks configured, e.g., `["pre-commit", "commit-msg"]`), `evidence`

---

## Signal: `naming_conventions`

**Purpose:** Consistent naming reduces cognitive load for agents parsing code.

**Detection:**

1. Search for linter rules enforcing naming:
   - ESLint: `@typescript-eslint/naming-convention` rule in config
   - golangci-lint: `revive` linter with `exported` rule, or `stylecheck` linter
   - Clippy: `module_name_repetitions`, `similar_names` lints
   - Ruff: `N` rules (pep8-naming) in `[tool.ruff.lint.select]`
   - pylint: `invalid-name` check enabled
2. Search documentation:
   - `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `STYLEGUIDE.md` — grep for "naming", "convention", "camelCase", "snake_case", "PascalCase"
3. Check `.editorconfig` for naming patterns

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `large_file_detection`

**Purpose:** Prevents large binaries from bloating the repository.

**Detection:**

1. Search for files:
   - `.gitattributes` — grep for `filter=lfs` or `lfs`
   - `.lfsconfig`
2. Search pre-commit hooks:
   - `.pre-commit-config.yaml` — check for `check-added-large-files` hook
3. Search CI workflows:
   - Steps checking file sizes, `git lfs`, or `check-added-large-files`

**Output fields:** `found`, `tools`, `config_files`, `lfs_configured` (true if Git LFS patterns found), `evidence`

---

## Signal: `code_modularization`

**Purpose:** Module boundary enforcement prevents tangled dependencies.

**Detection:**

1. Go: check for `internal/` directory (enforces import boundaries)
2. TypeScript/JavaScript:
   - `package.json` or config with `eslint-plugin-boundaries`, `eslint-plugin-import/no-restricted-paths`
   - Nx `project.json` with `depConstraints`
   - `.dependency-cruiser.js`, `.dependency-cruiser.cjs`
3. Rust: check for multiple crates in workspace (`Cargo.toml` with `[workspace]` and `members`)
4. Python: check for `__init__.py` files indicating package structure, or `py.typed` marker files
5. Java/Kotlin: check for module-info.java or multi-module Gradle/Maven setup

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `cyclomatic_complexity`

**Purpose:** Complexity limits prevent overly complex functions that agents struggle to modify.

**Detection:**

1. Search linter configs for complexity rules:
   - ESLint: `complexity` rule, `max-depth`, `max-nested-callbacks`
   - golangci-lint: `gocyclo`, `cyclop`, `gocognit` linters enabled
   - Ruff: `C901` rule in `[tool.ruff.lint.select]`
   - pylint: `max-complexity` setting
   - Clippy: `cognitive_complexity` lint
   - `.codeclimate.yml`: complexity checks
2. Search for standalone tools:
   - `radon` in Python dependencies
   - `gocyclo` in Go tools
   - SonarQube `sonar-project.properties`

**Output fields:** `found`, `tools`, `config_files`, `thresholds` (e.g., `{"max_complexity": 10}`), `evidence`

---

## Signal: `dead_code_detection`

**Purpose:** Unused code confuses agents and inflates context.

**Detection:**

1. Search for tools:
   - `package.json`: `knip`, `ts-prune`, `unimported`
   - `pyproject.toml`: `vulture`, `autoflake`
   - Ruff: `F401` (unused imports), `F841` (unused variables) in lint rules
   - golangci-lint: `unused`, `deadcode`, `structcheck`, `varcheck` linters
   - Rust: compiler warnings for `dead_code` (check for `#[deny(dead_code)]`)
   - staticcheck: `U1000` check
2. Search CI workflows:
   - Steps running `knip`, `vulture`, `deadcode`, `ts-prune`, `autoflake`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `duplicate_detection`

**Purpose:** Duplicate code creates maintenance burden and inconsistency.

**Detection:**

1. Search for tools:
   - `package.json`: `jscpd`
   - `.jscpd.json` config file
   - golangci-lint: `dupl` linter enabled
   - SonarQube or CodeClimate configs
   - PMD CPD in Java build configs
2. Search CI workflows:
   - Steps running `jscpd`, `cpd`, `dupl`, `simian`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `tech_debt_tracking`

**Purpose:** Systematic tracking of TODOs and FIXMEs prevents tech debt accumulation.

**Detection:**

1. Search CI workflows for:
   - Steps scanning for `TODO`, `FIXME`, `HACK`, `XXX` markers
   - Tools: `leasot`, `todo-check`, `grep -r TODO` in CI
   - SonarQube `sonar.issue.ignore` patterns
2. Search for tracking files:
   - `TODO.md`, `TECH_DEBT.md`, `docs/tech-debt/`
3. Search pre-commit hooks:
   - Hooks that check for TODO markers

**Output fields:** `found`, `tools`, `config_files`, `ci_scanning` (true if CI automates TODO scanning), `evidence`

---

## Signal: `n_plus_one_detection`

**Purpose:** N+1 query detection prevents database performance issues.

**Detection:**

1. Search dependency manifests:
   - Ruby Gemfile: `bullet`
   - Python: `nplusone`, `django-auto-prefetch`, `django-query-inspector`
   - `package.json`: `@sentry/node` with performance monitoring
2. Search for Prisma query warnings in config
3. If no database layer detected (no ORM, no database dependencies), set `found: null` with `skipped_reason`

**Output fields:** `found` (or `null` if skipped), `tools`, `config_files`, `skipped_reason` (if applicable), `evidence`
