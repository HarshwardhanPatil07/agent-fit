# Pillar 4: Documentation

Scan for 8 signals that measure whether documentation supports autonomous agent workflows.

---

## Signal: `readme`

**Purpose:** README is the primary entry point for understanding a project.

**Detection:**

1. Search for README files:
   - `README.md`, `README.rst`, `README.txt`, `README`, `readme.md`
   - For monorepos, also check per-package READMEs
2. Analyze README content quality (read the file):
   - `has_install`: grep for "install", "setup", "getting started", "quick start" (case-insensitive)
   - `has_usage`: grep for "usage", "example", "how to use", code blocks with commands
   - `has_contributing`: grep for "contribut", "CONTRIBUTING.md" reference
3. Get file size: `wc -c README.md`

**Output fields:** `found`, `tools`, `config_files`, `size_bytes`, `has_install`, `has_usage`, `has_contributing`, `evidence`

---

## Signal: `agents_md`

**Purpose:** AGENTS.md / CLAUDE.md provides agent-specific instructions for autonomous operation.

**Detection:**

1. Search for agent instruction files:
   - `AGENTS.md`, `agents.md`
   - `CLAUDE.md`, `claude.md`
   - `.claude/CLAUDE.md`
   - `.github/copilot-instructions.md`
   - `.cursorrules`
   - `.aider.conf.yml`
2. If found, briefly summarize content (does it include build commands, test commands, project conventions?)

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `agents_md_validation`

**Purpose:** CI validation ensures AGENTS.md commands remain accurate over time.

**Detection:**

1. Search CI workflows for:
   - Steps that parse or execute commands from AGENTS.md / CLAUDE.md
   - Steps that verify documentation accuracy
   - Tests that validate documented commands still work
2. Search for scripts:
   - `scripts/validate-docs.sh`, `scripts/check-agents-md.sh`
   - CI jobs named `verify-docs`, `validate-agents`

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `doc_freshness`

**Purpose:** Stale documentation misleads agents into following outdated instructions.

**Detection:**

1. Check modification dates of key docs:
   - `git log -1 --format="%ai" -- README.md` (get last modification date)
   - Same for `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`
2. Calculate days since last modification
3. Pass if any key doc was modified within 180 days

**Output fields:** `found`, `tools`, `config_files`, `last_modified_days` (days since most recently modified key doc), `evidence`

---

## Signal: `automated_doc_generation`

**Purpose:** Auto-generated docs stay in sync with code.

**Detection:**

1. Search for doc generation tools:
   - `package.json`: `typedoc`, `jsdoc`, `@compodoc/compodoc`, `storybook`
   - `pyproject.toml` / `requirements.txt`: `sphinx`, `mkdocs`, `pdoc`, `pydoc`
   - `docs/conf.py` (Sphinx), `mkdocs.yml` (MkDocs)
   - `doc.go`, `godoc` patterns (Go)
   - `Doxyfile`, `doxygen.cfg` (C/C++)
   - `cargo doc` in Makefile or CI (Rust)
   - `jazzy` for Swift
2. Search for doc generation configs:
   - `typedoc.json`, `typedoc.config.js`
   - `mkdocs.yml`
   - `docs/conf.py` (Sphinx)
   - `Doxyfile`
   - `.storybook/`
3. Search CI for doc generation:
   - Steps running `typedoc`, `sphinx-build`, `mkdocs build`, `godoc`, `cargo doc`
   - Steps deploying docs to GitHub Pages

**Output fields:** `found`, `tools`, `config_files`, `evidence`

---

## Signal: `api_schema_docs`

**Purpose:** API schemas enable agents to interact with services programmatically.

**Detection:**

1. Search for API schema files:
   - OpenAPI/Swagger: `openapi.yaml`, `openapi.json`, `swagger.yaml`, `swagger.json`, `api-spec.*`, `docs/api/`
   - GraphQL: `schema.graphql`, `schema.gql`, `*.graphqls`, `graphql/schema/`
   - Protocol Buffers: `*.proto` files, `proto/` directory
   - gRPC: `*.proto` with `service` definitions
   - AsyncAPI: `asyncapi.yaml`, `asyncapi.json`
   - JSON Schema: `schemas/`, `json-schema/`
   - WSDL: `*.wsdl`
2. Search for API doc tools:
   - `package.json`: `swagger-jsdoc`, `@nestjs/swagger`, `tsoa`
   - Python: `drf-spectacular`, `flask-restx`, `flasgger`, `fastapi` (auto-generates OpenAPI)

**Output fields:** `found`, `tools`, `config_files`, `schema_types` (list like `["openapi", "graphql", "protobuf"]`), `evidence`

---

## Signal: `service_flow_documented`

**Purpose:** Architecture diagrams help agents understand system boundaries.

**Detection:**

1. Search for diagram files:
   - PlantUML: `*.puml`, `*.plantuml`, `*.pu`
   - Mermaid: grep for `\`\`\`mermaid` in `.md` files
   - Draw.io: `*.drawio`, `*.drawio.svg`
   - Architecture docs: `docs/architecture/`, `docs/design/`, `ARCHITECTURE.md`, `docs/adr/`
   - Image diagrams: `docs/*.png`, `docs/*.svg` with architecture-related names
   - D2: `*.d2` files
2. Search for ADR (Architecture Decision Records):
   - `docs/adr/`, `docs/decisions/`, `adr/`

**Output fields:** `found`, `tools`, `config_files`, `diagram_types` (list like `["mermaid", "plantuml", "drawio"]`), `evidence`

---

## Signal: `skills_directory`

**Purpose:** Reusable agent skill definitions enable structured agent interaction.

**Detection:**

1. Search for skill directories:
   - `.claude/skills/` and any subdirectories
   - `.factory/skills/` and any subdirectories
   - `.claude/commands/` (legacy format)
2. Count skill files found
3. List skill names

**Output fields:** `found`, `tools`, `config_files`, `skill_count`, `evidence`
