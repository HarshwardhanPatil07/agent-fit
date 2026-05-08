# Pillar 7: Security and Governance

Scan for 11 signals that measure security posture and access governance.

---

## Signal: `branch_protection`

**Purpose:** Branch protection prevents unauthorized changes to main branches.

**Detection (remote only — requires GitHub API):**

1. Query GitHub API:
   - `gh api repos/{owner}/{repo}/branches/{default_branch}/protection`
   - If 403/404, record `api_accessible: false` and the error message
2. If accessible, extract rules:
   - `required_pull_request_reviews` (required reviewers count)
   - `required_status_checks` (CI checks that must pass)
   - `enforce_admins` (whether admins are exempt)
   - `restrictions` (push restrictions)
3. Also check for rulesets:
   - `gh api repos/{owner}/{repo}/rulesets`
4. For local scans, set `found: null` with explanation

**Output fields:** `found` (null for local or API-inaccessible), `tools`, `config_files`, `api_accessible`, `rules` (object with extracted rules or `null`), `error` (API error message or `null`), `evidence`

---

## Signal: `codeowners`

**Purpose:** CODEOWNERS defines who reviews what, enabling structured code review.

**Detection:**

1. Search for CODEOWNERS files:
   - `CODEOWNERS`, `.github/CODEOWNERS`, `docs/CODEOWNERS`
2. If found, analyze content:
   - Count number of team/individual assignments
   - Check if it covers key directories

**Output fields:** `found`, `tools`, `config_files`, `team_count` (number of distinct teams/individuals in CODEOWNERS), `evidence`

---

## Signal: `secret_scanning`

**Purpose:** Secret scanning detects credentials before they reach production.

**Detection (remote only — requires GitHub API):**

1. Check repository security settings:
   - `gh api repos/{owner}/{repo}` — check `security_and_analysis.secret_scanning.status`
   - Or: `gh api repos/{owner}/{repo}/secret-scanning/alerts?per_page=1`
   - If 403/404, record `api_accessible: false`
2. Search for local secret scanning tools:
   - `.gitleaks.toml`, `gitleaks.toml` (gitleaks config)
   - `.trufflehog.yml` or `trufflehog` in CI
   - `.pre-commit-config.yaml` — check for `detect-secrets`, `gitleaks`, `trufflehog` hooks
   - `git-secrets` references in CI
3. Search CI workflows:
   - Steps running `gitleaks`, `trufflehog`, `detect-secrets`, `git-secrets`

**Output fields:** `found` (null if API-only and inaccessible), `tools`, `config_files`, `api_accessible`, `enabled` (null if unknown), `error`, `evidence`

---

## Signal: `secrets_management`

**Purpose:** Secrets should use vaults or managers, never hardcoded values.

**Detection:**

1. Search for vault/secrets manager references:
   - Dependency manifests: `aws-sdk` (Secrets Manager), `@azure/keyvault-secrets`, `@google-cloud/secret-manager`, `hashicorp/vault`
   - Config files: `vault.hcl`, `.vault-token`, `vault-config/`
   - CI: `hashicorp/vault-action`, `aws-actions/configure-aws-credentials`
   - SOPS: `.sops.yaml`, `sops.yaml`
   - `chamber` references
   - `doppler.yaml`
2. Search source code for secrets patterns:
   - `grep -rl "SecretsManager\|SecretClient\|vault\.Read\|os\.Getenv.*SECRET\|os\.Getenv.*KEY\|os\.Getenv.*TOKEN" --include="*.go" --include="*.py" --include="*.ts" --include="*.js" . | head -5`
3. Check CI for secrets usage:
   - GitHub Actions `secrets.*` references (these use GitHub's built-in secrets management)

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `dependency_update_automation`

**Purpose:** Automated dependency updates reduce security debt.

**Detection:**

1. Search for bot configs:
   - `.github/dependabot.yml`, `.github/dependabot.yaml`
   - `renovate.json`, `renovate.json5`, `.renovaterc`, `.renovaterc.json`
   - `.github/renovate.json`, `.github/renovate.json5`
2. Check for recent bot PRs (remote only):
   - `gh pr list --author "dependabot[bot]" --state all --limit 3`
   - `gh pr list --author "renovate[bot]" --state all --limit 3`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `gitignore_comprehensive`

**Purpose:** A thorough .gitignore prevents sensitive files from entering the repo.

**Detection:**

1. Check for `.gitignore` at repo root
2. If found, analyze coverage:
   - `covers_env`: grep for `.env`, `*.env`, `.env.*` patterns
   - `covers_credentials`: grep for `credentials`, `*.pem`, `*.key`, `*.p12`, `*.jks`, `secrets`
   - `covers_build`: grep for `node_modules`, `dist/`, `build/`, `target/`, `__pycache__`, `*.pyc`, `vendor/`, `bin/`
   - Also check for IDE configs: `.idea/`, `.vscode/`, `*.swp`, `*.swo`
3. Check for additional `.gitignore` files in subdirectories

**Output fields:** `found`, `tools`, `config_files`, `covers_env`, `covers_credentials`, `covers_build`, `evidence`

---

## Signal: `automated_security_review`

**Purpose:** Security scanning in CI catches vulnerabilities before merge.

**Detection:**

1. Search CI workflows for security tools:
   - CodeQL: `github/codeql-action/analyze`, `github/codeql-action/init`
   - Snyk: `snyk/actions`, `snyk test`, `snyk monitor`
   - Trivy: `aquasecurity/trivy-action`
   - Semgrep: `returntocorp/semgrep-action`, `semgrep ci`
   - SAST: `sast` in CI step names
   - Bandit: `bandit` in CI (Python)
   - gosec: `securego/gosec` in CI (Go)
   - Brakeman: `brakeman` in CI (Ruby)
   - cargo-audit: `cargo audit` in CI (Rust)
2. Search for security scanning configs:
   - `.github/codeql/`, `codeql-config.yml`
   - `.snyk`
   - `.semgrep.yml`, `.semgrep/`
   - `.trivyignore`
   - `bandit.yaml`, `.bandit`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `log_scrubbing`

**Purpose:** Sensitive data must be redacted from logs before output.

**Detection:**

1. Search source code for redaction patterns:
   - `grep -rn "redact\|sanitize\|mask\|scrub\|SafeValue\|filter_sensitive\|REDACTED\|\\*\\*\\*" --include="*.go" --include="*.py" --include="*.ts" --include="*.js" --include="*.rs" --include="*.rb" . | head -10`
2. Search for log filtering configs:
   - Log formatter configurations that filter fields
   - Middleware that strips sensitive headers
3. Search for redaction utilities:
   - Files named `redact.*`, `sanitize.*`, `scrub.*`, `filter.*`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `pii_handling`

**Purpose:** Personal data must be classified and handled per privacy requirements.

**Detection:**

1. Search source code for PII handling patterns:
   - `grep -rn "pii\|personal.data\|data.classification\|gdpr\|ccpa\|anonymize\|pseudonymize\|encrypt.*email\|encrypt.*name" --include="*.go" --include="*.py" --include="*.ts" --include="*.js" -i . | head -10`
2. Search for privacy-related files:
   - `PRIVACY.md`, `docs/privacy/`, `docs/data-handling/`
   - Data classification schemas or policies
3. Search for data protection libraries:
   - Encryption helpers, tokenization utilities

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `dast_scanning`

**Purpose:** Dynamic application security testing finds runtime vulnerabilities.

**Detection:**

1. Search CI workflows for DAST tools:
   - OWASP ZAP: `zaproxy/action-*`, `zap-` in CI step names
   - Burp Suite: `burp` in CI
   - Nuclei: `projectdiscovery/nuclei-action`
   - DAST: `dast` in CI step names
2. Search for DAST configs:
   - `zap-config.yml`, `.zap/`
   - `nuclei-templates/`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `privacy_compliance`

**Purpose:** Privacy requirements (GDPR, CCPA) are enforced via tooling.

**Detection:**

1. Search for compliance configs:
   - Data retention policies in configs
   - Consent management libraries in dependencies
   - Cookie consent: `cookie-consent`, `cookieconsent`, `@osano/cookieconsent`
   - GDPR tools: `gdpr`, `data-privacy` in configs or deps
2. Search for compliance documentation:
   - `PRIVACY_POLICY.md`, `docs/compliance/`, `docs/gdpr/`
   - Data processing agreements

**Output fields:** `found`, `tools`, `config_files`, `evidence`
