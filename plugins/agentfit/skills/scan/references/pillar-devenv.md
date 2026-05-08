# Pillar 5: Development Environment

Scan for 5 signals that measure reproducibility and onboarding ease.

---

## Signal: `devcontainer`

**Purpose:** Dev containers provide reproducible environments, eliminating "works on my machine" issues.

**Detection:**

1. Search for devcontainer configs:
   - `.devcontainer/devcontainer.json`
   - `.devcontainer.json` (root-level)
   - `.devcontainer/docker-compose.yml`
   - `.devcontainer/Dockerfile`
2. If found, extract key info:
   - Base image (from `image` or `build.dockerfile`)
   - Features listed
   - Extensions listed
   - Post-create commands

**Output fields:** `found`, `tools`, `config_files`, `base_image` (if detectable), `evidence`

---

## Signal: `devcontainer_runnable`

**Purpose:** A devcontainer config that doesn't build is worse than none.

**Detection:**

This signal requires a container runtime and is not tested during scanning. Always set to `null` with `skipped_reason`.

**Output fields:** `found` (always `null`), `tools`, `config_files`, `skipped_reason` ("Requires container runtime to verify"), `evidence`

---

## Signal: `env_template`

**Purpose:** Environment variable templates prevent configuration guesswork.

**Detection:**

1. Search for env template files:
   - `.env.example`, `.env.sample`, `.env.template`, `.env.development`, `.env.local.example`
   - `.envrc.example`, `.envrc.sample`
   - `env.example`, `env.sample`
2. Search documentation for env var documentation:
   - `README.md`, `CONTRIBUTING.md`, `docs/setup.md` — grep for "environment variable", "env var", "ENV", "configuration"
   - `CLAUDE.md`, `AGENTS.md` — grep for environment setup instructions
3. If found, count variables:
   - `grep -c "=" .env.example` (approximate count)

**Output fields:** `found`, `tools`, `config_files`, `variable_count` (approximate number of env vars defined), `evidence`

---

## Signal: `local_services_setup`

**Purpose:** Containerized local dependencies eliminate manual service installation.

**Detection:**

1. Search for docker-compose files:
   - `docker-compose.yml`, `docker-compose.yaml`
   - `docker-compose.dev.yml`, `docker-compose.development.yml`
   - `docker-compose.override.yml`
   - `compose.yml`, `compose.yaml` (Docker Compose V2 format)
2. If found, extract service names:
   - Parse `services:` section to list service names (postgres, redis, elasticsearch, minio, etc.)
3. Search for alternative local service setups:
   - `Tiltfile` (Tilt)
   - `skaffold.yaml` (Skaffold)
   - `devspace.yaml` (DevSpace)
   - `Procfile`, `Procfile.dev` (Foreman/Hivemind)

**Output fields:** `found`, `tools`, `config_files`, `services` (list of service names), `evidence`

---

## Signal: `database_schema`

**Purpose:** Schema management enables agents to understand and modify data models.

**Detection:**

1. Search for migration directories:
   - `db/migrate/`, `db/migrations/`
   - `migrations/`, `migration/`
   - `alembic/`, `alembic.ini`
   - `prisma/`, `prisma/schema.prisma`
   - `drizzle/`, `drizzle.config.ts`
   - `sql/migrations/`, `sql/schema/`
   - `flyway/`, `flyway.conf`
   - `liquibase/`, `liquibase.properties`
   - `knex/migrations/`, `knexfile.js`, `knexfile.ts`
   - `typeorm/migrations/`
   - `sequelize/migrations/`, `.sequelizerc`
   - `sea-orm/migration/`
   - `goose/`, `atlas.hcl`
   - `ent/schema/` (Go Ent)
   - `sqlc.yaml`, `sqlc.yml`
2. Search for schema files:
   - `schema.prisma`, `schema.sql`, `schema.graphql`
   - `models.py` (Django/SQLAlchemy)
   - `*.entity.ts` (TypeORM)
3. Count migration files if migration directory found

**Output fields:** `found`, `tools`, `config_files`, `migration_tool` (e.g., `"alembic"`, `"prisma"`, `"goose"`), `migration_count`, `evidence`
