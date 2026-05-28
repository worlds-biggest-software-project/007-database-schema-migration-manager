# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Database Schema Migration Manager · Created: 2026-05-11

## Philosophy

This model uses a pragmatic hybrid approach: core structural fields that are always present and always queried are stored as typed relational columns, while variable, engine-specific, or rapidly-evolving fields are stored in JSONB columns. This is the "best of both worlds" approach that PostgreSQL's JSONB capabilities make uniquely powerful.

The key insight driving this design is that a database migration manager must support multiple database engines (PostgreSQL, MySQL, SQLite, SQL Server, Oracle, CockroachDB), each with different DDL syntax, schema introspection output, type systems, and migration capabilities. A fully normalized model would require either a massive union of all possible engine-specific columns (most of which would be NULL for any given engine) or a sprawling hierarchy of engine-specific subtables. JSONB columns absorb this variability elegantly.

This pattern is widely adopted in production SaaS platforms. Bytebase, for example, supports 20+ database engines through a unified data model with engine-specific configuration stored as structured JSON. The Prisma schema definition language similarly uses a declarative core with engine-specific attributes stored as metadata.

**Best for:** Rapid MVP development where the exact shape of engine-specific data will evolve quickly; multi-database platforms where the variability between PostgreSQL, MySQL, and Oracle configurations is too high for a single normalized schema; teams that want to ship fast while maintaining strong query capability on the core fields.

**Trade-offs:**
- (+) Fewer tables (~15) — dramatically simpler schema than the normalized approach
- (+) Engine-specific fields don't require schema migrations to add or change
- (+) JSONB GIN indexes provide fast queries on variable fields
- (+) PostgreSQL's JSONB operators enable rich querying within the JSON structure
- (+) Natural fit for multi-database engine support with varying metadata
- (+) Faster time-to-MVP — less upfront schema design needed
- (-) JSONB fields lack referential integrity — no FK constraints inside JSON
- (-) Type safety within JSONB depends on application-layer validation (JSON Schema)
- (-) JSONB columns can become "junk drawers" without disciplined documentation
- (-) Complex JSONB queries can be slower than relational JOINs for deeply nested structures
- (-) Harder to enforce NOT NULL or CHECK constraints on fields inside JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JSON Schema (IETF) | Application-layer validation of JSONB column contents; documented inline as comments |
| ISO/IEC 27001 | `activity_log` JSONB column on key tables + `audit_events` table for compliance |
| OpenGitOps v1.0 | `desired_state` JSONB column stores the full declared schema in engine-native format |
| DORA Metrics | Execution timing fields on `migration_runs` enable metric derivation |
| OpenAPI 3.1 | API request/response schemas match the JSONB structures, validated by JSON Schema |
| Semantic Versioning 2.0.0 | Migration ordering via `version` column; declaration versioning via `revision` |

---

## Core Platform Tables

```sql
-- ============================================================
-- CORE PLATFORM (Relational core + JSONB for variable data)
-- ============================================================

CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'pro', 'enterprise')),

    -- JSONB: workspace-level settings that vary by plan and preference
    settings        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- settings example:
    -- {
    --   "default_approval_policy": "require_dba",
    --   "allowed_engines": ["postgresql", "mysql"],
    --   "notification_defaults": {
    --     "slack_webhook": "https://hooks.slack.com/...",
    --     "notify_on": ["migration_failed", "drift_detected"]
    --   },
    --   "security": {
    --     "require_mfa": true,
    --     "session_timeout_minutes": 480,
    --     "allowed_ip_ranges": ["10.0.0.0/8"]
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'developer'
                    CHECK (role IN ('owner', 'admin', 'dba', 'developer', 'viewer')),

    -- JSONB: identity provider details, notification preferences
    profile         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- profile example:
    -- {
    --   "idp": "github",
    --   "idp_subject": "12345",
    --   "avatar_url": "https://...",
    --   "notification_prefs": {
    --     "email_on_approval_request": true,
    --     "slack_dm_on_failure": true
    --   },
    --   "timezone": "America/New_York"
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, email)
);

CREATE INDEX idx_members_workspace ON members(workspace_id);
CREATE INDEX idx_members_email ON members(email);
```

---

## Projects & Targets

```sql
-- ============================================================
-- PROJECTS & DATABASE TARGETS
-- ============================================================

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,

    -- JSONB: VCS integration, CI/CD configuration, project-level settings
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- config example:
    -- {
    --   "vcs": {
    --     "provider": "github",
    --     "repo_url": "https://github.com/org/repo",
    --     "branch": "main",
    --     "migration_path": "db/migrations/",
    --     "auto_detect_migrations": true
    --   },
    --   "ci": {
    --     "lint_on_pr": true,
    --     "block_destructive": true,
    --     "require_rollback_script": false
    --   },
    --   "naming_convention": "timestamp",
    --   "supported_engines": ["postgresql", "mysql"]
    -- }

    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE TABLE targets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,          -- 'development', 'staging', 'production'
    tier            VARCHAR(50) NOT NULL DEFAULT 'development'
                    CHECK (tier IN ('development', 'staging', 'production')),
    promotion_order INTEGER NOT NULL DEFAULT 0,
    engine          VARCHAR(50) NOT NULL
                    CHECK (engine IN ('postgresql', 'mysql', 'sqlite', 'sqlserver',
                           'oracle', 'cockroachdb', 'mariadb')),

    -- JSONB: engine-specific connection details and capabilities
    -- This is where the hybrid model shines — each engine has different
    -- connection parameters, capabilities, and introspection methods
    connection      JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- PostgreSQL example:
    -- {
    --   "host": "db.example.com",
    --   "port": 5432,
    --   "database": "myapp_prod",
    --   "schema": "public",
    --   "ssl_mode": "verify-full",
    --   "secret_ref": "vault://secret/db/prod",
    --   "pool_size": 5,
    --   "statement_timeout_ms": 30000
    -- }
    --
    -- MySQL example:
    -- {
    --   "host": "mysql.example.com",
    --   "port": 3306,
    --   "database": "myapp_prod",
    --   "charset": "utf8mb4",
    --   "secret_ref": "vault://secret/db/mysql-prod",
    --   "ssl_ca": "/path/to/ca.pem"
    -- }
    --
    -- SQLite example:
    -- {
    --   "path": "/data/app.db",
    --   "journal_mode": "wal"
    -- }

    -- JSONB: approval and policy configuration for this target
    policies        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- policies example:
    -- {
    --   "requires_approval": true,
    --   "min_approvals": 2,
    --   "required_roles": ["dba", "admin"],
    --   "auto_approve_risk": ["low"],
    --   "maintenance_window": {
    --     "day": "sunday",
    --     "start": "02:00",
    --     "end": "06:00",
    --     "timezone": "UTC"
    --   },
    --   "max_lock_wait_seconds": 30,
    --   "expand_contract_enabled": true
    -- }

    current_version VARCHAR(50),                   -- last applied migration version
    last_connected_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name)
);

CREATE INDEX idx_projects_workspace ON projects(workspace_id);
CREATE INDEX idx_targets_project ON targets(project_id);
CREATE INDEX idx_targets_engine ON targets(engine);
```

---

## Schema State & Migrations

```sql
-- ============================================================
-- SCHEMA STATE & MIGRATIONS
-- ============================================================

CREATE TABLE schema_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_id       UUID NOT NULL REFERENCES targets(id) ON DELETE CASCADE,
    snapshot_type   VARCHAR(50) NOT NULL
                    CHECK (snapshot_type IN ('declared', 'actual', 'diff')),

    -- JSONB: the full schema representation in a structured format
    -- This replaces the need for separate schema_objects table
    schema_data     JSONB NOT NULL,
    -- schema_data example (declared or actual):
    -- {
    --   "tables": {
    --     "users": {
    --       "columns": {
    --         "id": {"type": "uuid", "nullable": false, "default": "gen_random_uuid()", "primary_key": true},
    --         "email": {"type": "varchar(320)", "nullable": false, "unique": true},
    --         "name": {"type": "varchar(255)", "nullable": false},
    --         "created_at": {"type": "timestamptz", "nullable": false, "default": "now()"}
    --       },
    --       "indexes": {
    --         "idx_users_email": {"columns": ["email"], "unique": true}
    --       },
    --       "constraints": {
    --         "users_pkey": {"type": "primary_key", "columns": ["id"]}
    --       }
    --     },
    --     "orders": { ... }
    --   },
    --   "enums": {
    --     "order_status": {"values": ["pending", "shipped", "delivered"]}
    --   },
    --   "extensions": ["uuid-ossp", "pgcrypto"],
    --   "functions": { ... }
    -- }
    --
    -- diff example:
    -- {
    --   "added": {"tables": ["new_table"], "columns": ["users.phone"]},
    --   "removed": {"columns": ["users.legacy_id"]},
    --   "modified": {"columns": {"users.email": {"old_type": "varchar(255)", "new_type": "varchar(320)"}}}
    -- }

    content_hash    VARCHAR(64) NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    captured_by     UUID REFERENCES members(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE migrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,
    description     VARCHAR(500) NOT NULL,
    migration_type  VARCHAR(50) NOT NULL DEFAULT 'versioned'
                    CHECK (migration_type IN ('versioned', 'repeatable', 'undo', 'baseline')),
    source          VARCHAR(50) NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual', 'declarative_diff', 'ai_generated',
                           'import_flyway', 'import_liquibase')),

    -- Core migration SQL (always present)
    sql_up          TEXT NOT NULL,
    sql_down        TEXT,
    checksum        VARCHAR(64) NOT NULL,

    -- JSONB: analysis results, risk assessment, and AI-generated metadata
    analysis        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- analysis example:
    -- {
    --   "risk_level": "high",
    --   "is_destructive": true,
    --   "risk_factors": [
    --     {"type": "drop_column", "object": "users.legacy_id", "severity": "high"},
    --     {"type": "lock_risk", "object": "users", "estimated_lock_ms": 5000, "row_count": 1500000}
    --   ],
    --   "affected_objects": [
    --     {"type": "table", "schema": "public", "name": "users", "operation": "alter"},
    --     {"type": "column", "schema": "public", "name": "users.legacy_id", "operation": "drop"}
    --   ],
    --   "impact": {
    --     "risk_score": 0.75,
    --     "orm_references": [
    --       {"file": "src/models/user.ts", "line": 42, "type": "prisma_field"},
    --       {"file": "src/api/users.ts", "line": 108, "type": "raw_query"}
    --     ],
    --     "downstream_conflicts": [],
    --     "recommendation": "Use expand-contract: add new column, backfill, switch reads, drop old"
    --   },
    --   "ai_confidence": 0.92,
    --   "analyzed_at": "2026-05-11T10:00:00Z"
    -- }

    -- JSONB: engine-specific migration options
    engine_options  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- PostgreSQL example:
    -- {
    --   "lock_timeout_ms": 5000,
    --   "statement_timeout_ms": 30000,
    --   "use_concurrent_index": true,
    --   "expand_contract": {
    --     "phase": "expand",
    --     "paired_migration_id": "uuid-of-contract-phase"
    --   }
    -- }
    --
    -- MySQL example:
    -- {
    --   "algorithm": "INPLACE",
    --   "lock": "NONE",
    --   "use_gh_ost": true
    -- }

    authored_by     UUID NOT NULL REFERENCES members(id),
    commit_sha      VARCHAR(40),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

CREATE INDEX idx_schema_snapshots_target ON schema_snapshots(target_id);
CREATE INDEX idx_schema_snapshots_type ON schema_snapshots(snapshot_type);
CREATE INDEX idx_migrations_project ON migrations(project_id);
CREATE INDEX idx_migrations_version ON migrations(project_id, version);
-- GIN index enables queries like "find all migrations affecting table X"
CREATE INDEX idx_migrations_analysis ON migrations USING GIN (analysis jsonb_path_ops);
```

---

## Execution, Approvals & Drift

```sql
-- ============================================================
-- EXECUTION, APPROVALS & DRIFT (combined for simplicity)
-- ============================================================

CREATE TABLE migration_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_id    UUID NOT NULL REFERENCES migrations(id),
    target_id       UUID NOT NULL REFERENCES targets(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'awaiting_approval', 'approved',
                           'rejected', 'running', 'success', 'failed',
                           'rolled_back', 'skipped')),
    executed_sql    TEXT,
    execution_time_ms INTEGER,
    executed_by     UUID REFERENCES members(id),
    executed_at     TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,

    -- JSONB: approval decisions, error details, rollback info
    -- Combines what would be 3 separate tables in a normalized model
    details         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- details example (with approvals):
    -- {
    --   "approvals": [
    --     {
    --       "reviewer_id": "uuid",
    --       "reviewer_name": "Jane DBA",
    --       "decision": "approve",
    --       "comment": "LGTM — verified expand-contract is safe",
    --       "decided_at": "2026-05-11T09:00:00Z"
    --     }
    --   ],
    --   "required_approvals": 2,
    --   "approval_policy": "require_dba",
    --   "error": {
    --     "message": "ERROR: column \"legacy_id\" does not exist",
    --     "detail": "...",
    --     "sql_state": "42703"
    --   },
    --   "rollback": {
    --     "executed_at": "2026-05-11T10:15:00Z",
    --     "executed_by": "uuid",
    --     "sql": "ALTER TABLE users ADD COLUMN legacy_id ...",
    --     "success": true
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE drift_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_id       UUID NOT NULL REFERENCES targets(id),
    scan_type       VARCHAR(50) NOT NULL DEFAULT 'scheduled'
                    CHECK (scan_type IN ('scheduled', 'manual', 'ci_trigger', 'post_deploy')),
    status          VARCHAR(50) NOT NULL DEFAULT 'clean'
                    CHECK (status IN ('clean', 'drifted', 'error')),

    -- JSONB: all drift findings in a single structured document
    findings        JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- findings example:
    -- [
    --   {
    --     "object_type": "column",
    --     "schema": "public",
    --     "table": "users",
    --     "column": "email",
    --     "drift_type": "column_type_mismatch",
    --     "expected": "varchar(255)",
    --     "actual": "text",
    --     "severity": "warning",
    --     "is_resolved": false
    --   },
    --   {
    --     "object_type": "index",
    --     "schema": "public",
    --     "name": "idx_users_phone",
    --     "drift_type": "missing_in_declaration",
    --     "severity": "info",
    --     "probable_cause": {
    --       "type": "manual_hotfix",
    --       "correlated_commit": "abc123",
    --       "correlated_user": "ops-team"
    --     }
    --   }
    -- ]

    drift_count     INTEGER NOT NULL DEFAULT 0,
    scanned_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_migration_runs_migration ON migration_runs(migration_id);
CREATE INDEX idx_migration_runs_target ON migration_runs(target_id);
CREATE INDEX idx_migration_runs_status ON migration_runs(status);
CREATE INDEX idx_drift_reports_target ON drift_reports(target_id);
CREATE INDEX idx_drift_reports_status ON drift_reports(status) WHERE status = 'drifted';
-- GIN index for querying drift findings by object name
CREATE INDEX idx_drift_findings ON drift_reports USING GIN (findings jsonb_path_ops);
```

---

## Audit & Notifications

```sql
-- ============================================================
-- AUDIT & NOTIFICATIONS
-- ============================================================

CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id),
    actor_id        UUID REFERENCES members(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID NOT NULL,

    -- JSONB: action-specific details and before/after state
    context         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- context example:
    -- {
    --   "ip_address": "1.2.3.4",
    --   "user_agent": "...",
    --   "before": {"role": "developer"},
    --   "after": {"role": "dba"},
    --   "project_id": "uuid",
    --   "environment": "production"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notification_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,

    -- JSONB: flexible notification configuration
    config          JSONB NOT NULL,
    -- config example:
    -- {
    --   "channel": "slack",
    --   "webhook_url": "https://hooks.slack.com/...",
    --   "slack_channel": "#db-migrations",
    --   "events": ["migration_failed", "drift_detected", "approval_requested"],
    --   "filters": {
    --     "environments": ["production", "staging"],
    --     "risk_levels": ["high", "critical"]
    --   }
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_events_workspace ON audit_events(workspace_id);
CREATE INDEX idx_audit_events_action ON audit_events(action);
CREATE INDEX idx_audit_events_resource ON audit_events(resource_type, resource_id);
CREATE INDEX idx_audit_events_created ON audit_events(created_at);
CREATE INDEX idx_audit_events_context ON audit_events USING GIN (context jsonb_path_ops);
CREATE INDEX idx_notification_rules_project ON notification_rules(project_id);
```

---

## JSONB Query Examples

```sql
-- ============================================================
-- JSONB QUERY EXAMPLES
-- ============================================================

-- Find all migrations that affect a specific table
SELECT id, version, description
FROM migrations
WHERE analysis @> '{"affected_objects": [{"name": "users"}]}'::jsonb;

-- Find all high-risk migrations with ORM references
SELECT id, version, analysis->'impact'->>'risk_score' AS risk_score
FROM migrations
WHERE analysis @> '{"risk_level": "high"}'::jsonb
  AND jsonb_array_length(analysis->'impact'->'orm_references') > 0;

-- Find all unresolved drift findings for a specific table
SELECT id, findings
FROM drift_reports
WHERE target_id = :target_id
  AND status = 'drifted'
  AND findings @> '[{"table": "users", "is_resolved": false}]'::jsonb;

-- Find migrations with expand-contract configuration
SELECT id, version, engine_options->'expand_contract'->>'phase' AS phase
FROM migrations
WHERE engine_options ? 'expand_contract';

-- Get all approval decisions for a migration run
SELECT
    id,
    jsonb_array_elements(details->'approvals') AS approval
FROM migration_runs
WHERE migration_id = :migration_id;

-- Aggregate: count migrations by risk level using JSONB
SELECT
    analysis->>'risk_level' AS risk_level,
    COUNT(*) AS count
FROM migrations
WHERE project_id = :project_id
GROUP BY analysis->>'risk_level';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Platform | 2 | workspaces, members |
| Projects & Targets | 2 | projects, targets |
| Schema & Migrations | 3 | schema_snapshots, migrations |
| Execution & Drift | 2 | migration_runs, drift_reports |
| Audit & Notifications | 2 | audit_events, notification_rules |
| **Total** | **11** | Versus ~25 in the normalized model |

---

## Key Design Decisions

1. **`targets` instead of separate `environments` + `database_connections` tables** — a target is the combination of an environment and its database connection. Engine-specific connection details (PostgreSQL SSL settings vs. MySQL charset vs. SQLite path) live in the `connection` JSONB column, absorbing cross-engine variability without engine-specific subtables.

2. **`analysis` JSONB on `migrations` replaces 3 normalized tables** — impact analysis results, risk assessment, and affected objects are stored in a single structured JSONB column. This eliminates `impact_analyses`, `impact_findings`, and `migration_affected_objects` from the normalized model. The GIN index on `analysis` enables performant queries.

3. **Approval decisions embedded in `migration_runs.details`** — rather than separate `approval_requests` and `approval_decisions` tables, the approval workflow state is stored as a JSONB array within the migration run. This trades referential integrity (no FK to reviewer) for simplicity and atomic reads ("give me a migration run with all its approvals in one query").

4. **`schema_snapshots` with full JSONB schema representation** — instead of a `schema_objects` table with one row per column/index/constraint, the entire schema is captured as a structured JSONB document. This enables full-schema diffing with a single read and aligns with how tools like Atlas and `pg_dump --format=json` represent schemas.

5. **`drift_reports.findings` as a JSONB array** — drift findings are always consumed as a group (the full drift report), not individually queried by finding ID. Storing them as a JSONB array within the report eliminates a junction table and enables the report to include AI-generated root-cause correlation inline.

6. **`engine_options` JSONB on `migrations`** — PostgreSQL-specific options (concurrent index creation, lock timeout) vs. MySQL-specific options (ALGORITHM, gh-ost) vs. Oracle-specific options are fundamentally different. A JSONB column absorbs this variability without requiring engine-specific columns or subtables.

7. **Audit events use JSONB `context` for before/after state** — the `context` column captures action-specific details, including before/after state diffs for state-changing operations. This provides SOC 2 / ISO 27001 auditability without a rigid column structure that would need to change with every new action type.

8. **11 tables vs. 25** — this model achieves roughly the same functionality as the normalized model with less than half the tables. The trade-off is that JSONB field contents are validated at the application layer (via JSON Schema) rather than by database constraints.

9. **GIN indexes on every JSONB column** — the `jsonb_path_ops` GIN index class supports the `@>` containment operator, which covers the majority of JSONB query patterns. This ensures that JSONB queries perform comparably to relational queries for the most common access patterns.

10. **Notification rules as JSONB configuration** — rather than separate tables for channels, subscriptions, and event filters, a single `notification_rules` table with a JSONB `config` column captures the full notification policy. This is appropriate because notification rules are read infrequently and always as a complete unit.
