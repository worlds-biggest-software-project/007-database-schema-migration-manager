# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Database Schema Migration Manager · Created: 2026-05-11

## Philosophy

This model follows the traditional normalized relational approach: every domain concept gets its own table with strong foreign key relationships, explicit junction tables for many-to-many relationships, and no denormalization. The schema is designed for data integrity first, with every relationship enforced at the database level.

This approach mirrors how Flyway and Liquibase track their own state (via `flyway_schema_history` and `DATABASECHANGELOG` tables), but extends it dramatically to cover the full lifecycle of a migration management platform — projects, environments, migrations, approvals, drift detection, impact analysis, and audit trails. The Flyway schema history table (columns: `installed_rank`, `version`, `description`, `type`, `script`, `checksum`, `installed_by`, `installed_on`, `execution_time`, `success`) serves as a reference for the migration execution tracking portion of this model.

**Best for:** Teams that value referential integrity, need complex cross-entity queries (e.g., "show me all migrations affecting column X across all projects in the last 30 days"), and operate in regulated environments where every relationship must be explicitly modeled and auditable.

**Trade-offs:**
- (+) Maximum data integrity — foreign keys prevent orphaned records
- (+) Complex queries are straightforward with JOINs across well-defined relationships
- (+) Easy to reason about for developers familiar with relational modeling
- (+) Strong alignment with SOC 2 / ISO 27001 audit requirements
- (-) Higher table count (~35+ tables) increases schema complexity
- (-) Schema evolution of the migration manager itself requires careful migration planning
- (-) Many-to-many junction tables add write overhead
- (-) Less flexible for jurisdiction-specific or database-engine-specific metadata that varies widely

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 27001 | Audit trail tables (`audit_logs`, `approval_decisions`) provide tamper-evident change history |
| ISO/IEC 25012 | Data quality characteristics enforced via NOT NULL constraints, CHECK constraints, and FK relationships |
| OpenGitOps v1.0 | `schema_declarations` table stores desired-state definitions; `migration_plans` computed from declared vs. actual |
| Semantic Versioning 2.0.0 | `migrations.version` follows SemVer for ordering; `schema_declarations.version` tracks declaration versions |
| DORA Metrics | `migration_executions` table captures `execution_time_ms` and `success` for change failure rate and MTTR calculation |
| OAuth 2.0 / OIDC | `users` table stores identity provider references; `api_tokens` table for programmatic access |
| OpenAPI 3.1 | API endpoint contracts align with the entity structure (one resource per table) |

---

## Identity & Access Management

```sql
-- ============================================================
-- IDENTITY & ACCESS MANAGEMENT
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free'
                    CHECK (plan_tier IN ('free', 'pro', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    identity_provider VARCHAR(50) NOT NULL DEFAULT 'local'
                    CHECK (identity_provider IN ('local', 'google', 'github', 'okta', 'saml')),
    idp_subject_id  VARCHAR(255),               -- external identity provider subject ID
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'admin', 'dba', 'developer', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    token_hash      VARCHAR(64) NOT NULL UNIQUE,  -- SHA-256 hash of the token
    scopes          TEXT[] NOT NULL DEFAULT '{}',   -- e.g., {'migrations:read', 'migrations:write'}
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_memberships_user ON organization_memberships(user_id);
CREATE INDEX idx_org_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_api_tokens_user ON api_tokens(user_id);
```

---

## Project & Environment Management

```sql
-- ============================================================
-- PROJECT & ENVIRONMENT MANAGEMENT
-- ============================================================

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    vcs_repo_url    TEXT,                          -- e.g., https://github.com/org/repo
    vcs_provider    VARCHAR(50)
                    CHECK (vcs_provider IN ('github', 'gitlab', 'bitbucket', 'azure_devops')),
    default_branch  VARCHAR(100) DEFAULT 'main',
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE project_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'developer'
                    CHECK (role IN ('owner', 'dba', 'developer', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, user_id)
);

CREATE TABLE environments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,         -- e.g., 'development', 'staging', 'production'
    slug            VARCHAR(100) NOT NULL,
    tier            VARCHAR(50) NOT NULL DEFAULT 'development'
                    CHECK (tier IN ('development', 'staging', 'production')),
    promotion_order INTEGER NOT NULL DEFAULT 0,    -- 0=dev, 1=staging, 2=prod
    requires_approval BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

CREATE TABLE database_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    environment_id  UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
    engine          VARCHAR(50) NOT NULL
                    CHECK (engine IN ('postgresql', 'mysql', 'sqlite', 'sqlserver', 'oracle', 'cockroachdb')),
    host            VARCHAR(255),
    port            INTEGER,
    database_name   VARCHAR(255) NOT NULL,
    schema_name     VARCHAR(255) DEFAULT 'public',
    connection_options JSONB NOT NULL DEFAULT '{}',
    -- Credentials stored in external secret manager; reference only
    secret_ref      VARCHAR(500),                  -- e.g., vault://secret/db/prod
    is_read_only    BOOLEAN NOT NULL DEFAULT false,
    last_connected_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_environments_project ON environments(project_id);
CREATE INDEX idx_db_connections_env ON database_connections(environment_id);
```

---

## Schema Declarations (Desired State)

```sql
-- ============================================================
-- SCHEMA DECLARATIONS (DESIRED STATE)
-- ============================================================

CREATE TABLE schema_declarations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,          -- SemVer: '1.0.0', '1.1.0'
    format          VARCHAR(20) NOT NULL DEFAULT 'sql'
                    CHECK (format IN ('sql', 'hcl', 'yaml', 'prisma', 'orm_model')),
    content         TEXT NOT NULL,                 -- the full schema declaration
    content_hash    VARCHAR(64) NOT NULL,          -- SHA-256 hash for integrity checking
    parent_version_id UUID REFERENCES schema_declarations(id),
    authored_by     UUID NOT NULL REFERENCES users(id),
    commit_sha      VARCHAR(40),                   -- git commit SHA if VCS-linked
    is_current      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

CREATE TABLE schema_objects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    declaration_id  UUID NOT NULL REFERENCES schema_declarations(id) ON DELETE CASCADE,
    object_type     VARCHAR(50) NOT NULL
                    CHECK (object_type IN ('table', 'column', 'index', 'constraint',
                           'trigger', 'function', 'view', 'sequence', 'enum', 'extension')),
    schema_name     VARCHAR(255) NOT NULL DEFAULT 'public',
    object_name     VARCHAR(255) NOT NULL,
    parent_object_id UUID REFERENCES schema_objects(id), -- column -> table
    definition      TEXT NOT NULL,                 -- DDL fragment for this object
    attributes      JSONB NOT NULL DEFAULT '{}',   -- type-specific attributes
    -- Example attributes for a column:
    -- {"data_type": "varchar(255)", "nullable": false, "default": "gen_random_uuid()"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schema_declarations_project ON schema_declarations(project_id);
CREATE INDEX idx_schema_declarations_current ON schema_declarations(project_id) WHERE is_current = true;
CREATE INDEX idx_schema_objects_declaration ON schema_objects(declaration_id);
CREATE INDEX idx_schema_objects_type ON schema_objects(object_type, object_name);
```

---

## Migration Management

```sql
-- ============================================================
-- MIGRATION MANAGEMENT
-- ============================================================

CREATE TABLE migrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,          -- ordered version: 'V001', '20260511120000'
    description     VARCHAR(500) NOT NULL,
    migration_type  VARCHAR(50) NOT NULL DEFAULT 'versioned'
                    CHECK (migration_type IN ('versioned', 'repeatable', 'undo', 'baseline')),
    source          VARCHAR(50) NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual', 'declarative_diff', 'ai_generated', 'import')),
    -- Links to the desired-state declaration that generated this migration
    source_declaration_id UUID REFERENCES schema_declarations(id),
    sql_up          TEXT NOT NULL,                 -- forward migration SQL
    sql_down        TEXT,                          -- rollback SQL (NULL if irreversible)
    checksum        VARCHAR(64) NOT NULL,          -- SHA-256 of sql_up
    is_destructive  BOOLEAN NOT NULL DEFAULT false,-- true if DROP, ALTER TYPE, etc.
    risk_level      VARCHAR(20) NOT NULL DEFAULT 'low'
                    CHECK (risk_level IN ('low', 'medium', 'high', 'critical')),
    authored_by     UUID NOT NULL REFERENCES users(id),
    commit_sha      VARCHAR(40),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

CREATE TABLE migration_dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_id    UUID NOT NULL REFERENCES migrations(id) ON DELETE CASCADE,
    depends_on_id   UUID NOT NULL REFERENCES migrations(id) ON DELETE CASCADE,
    dependency_type VARCHAR(50) NOT NULL DEFAULT 'requires'
                    CHECK (dependency_type IN ('requires', 'conflicts_with', 'supersedes')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (migration_id, depends_on_id)
);

CREATE TABLE migration_affected_objects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_id    UUID NOT NULL REFERENCES migrations(id) ON DELETE CASCADE,
    object_type     VARCHAR(50) NOT NULL
                    CHECK (object_type IN ('table', 'column', 'index', 'constraint',
                           'trigger', 'function', 'view', 'sequence', 'enum')),
    schema_name     VARCHAR(255) NOT NULL DEFAULT 'public',
    object_name     VARCHAR(255) NOT NULL,
    operation       VARCHAR(50) NOT NULL
                    CHECK (operation IN ('create', 'alter', 'drop', 'rename', 'add_column',
                           'drop_column', 'alter_column', 'add_index', 'drop_index')),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_migrations_project ON migrations(project_id);
CREATE INDEX idx_migrations_version ON migrations(project_id, version);
CREATE INDEX idx_migration_deps ON migration_dependencies(migration_id);
CREATE INDEX idx_migration_affected ON migration_affected_objects(migration_id);
CREATE INDEX idx_migration_affected_object ON migration_affected_objects(object_type, object_name);
```

---

## Migration Execution & History

```sql
-- ============================================================
-- MIGRATION EXECUTION & HISTORY
-- ============================================================

-- Modeled after Flyway's flyway_schema_history and Liquibase's DATABASECHANGELOG
CREATE TABLE migration_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_id    UUID NOT NULL REFERENCES migrations(id),
    environment_id  UUID NOT NULL REFERENCES environments(id),
    connection_id   UUID NOT NULL REFERENCES database_connections(id),
    -- Flyway-compatible fields
    installed_rank  INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'success', 'failed',
                           'rolled_back', 'skipped')),
    executed_sql    TEXT NOT NULL,                 -- actual SQL that was run
    execution_time_ms INTEGER,                    -- for DORA metrics: change lead time
    checksum        VARCHAR(64) NOT NULL,
    executed_by     UUID NOT NULL REFERENCES users(id),
    executed_at     TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    error_detail    TEXT,
    rollback_execution_id UUID REFERENCES migration_executions(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Execution lock to prevent concurrent migrations (Liquibase DATABASECHANGELOGLOCK pattern)
CREATE TABLE migration_locks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    environment_id  UUID NOT NULL REFERENCES environments(id) UNIQUE,
    locked          BOOLEAN NOT NULL DEFAULT false,
    locked_by       UUID REFERENCES users(id),
    lock_granted_at TIMESTAMPTZ,
    lock_expires_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_executions_migration ON migration_executions(migration_id);
CREATE INDEX idx_executions_env ON migration_executions(environment_id);
CREATE INDEX idx_executions_status ON migration_executions(status);
CREATE INDEX idx_executions_at ON migration_executions(executed_at);
```

---

## Approval Workflows

```sql
-- ============================================================
-- APPROVAL WORKFLOWS
-- ============================================================

CREATE TABLE approval_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID REFERENCES environments(id) ON DELETE CASCADE,  -- NULL = project-wide
    name            VARCHAR(255) NOT NULL,
    min_approvals   INTEGER NOT NULL DEFAULT 1,
    required_roles  TEXT[] NOT NULL DEFAULT '{dba}', -- roles that can approve
    auto_approve_risk_levels TEXT[] DEFAULT '{low}', -- auto-approve low-risk changes
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_id    UUID NOT NULL REFERENCES migrations(id),
    environment_id  UUID NOT NULL REFERENCES environments(id),
    policy_id       UUID NOT NULL REFERENCES approval_policies(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'expired', 'auto_approved')),
    requested_by    UUID NOT NULL REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES approval_requests(id) ON DELETE CASCADE,
    reviewer_id     UUID NOT NULL REFERENCES users(id),
    decision        VARCHAR(20) NOT NULL
                    CHECK (decision IN ('approve', 'reject', 'request_changes')),
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_approval_requests_migration ON approval_requests(migration_id);
CREATE INDEX idx_approval_requests_status ON approval_requests(status);
CREATE INDEX idx_approval_decisions_request ON approval_decisions(request_id);
```

---

## Drift Detection

```sql
-- ============================================================
-- DRIFT DETECTION
-- ============================================================

CREATE TABLE drift_scans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    environment_id  UUID NOT NULL REFERENCES environments(id),
    connection_id   UUID NOT NULL REFERENCES database_connections(id),
    declaration_id  UUID REFERENCES schema_declarations(id),  -- declared state compared against
    status          VARCHAR(50) NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'clean', 'drifted', 'error')),
    scan_type       VARCHAR(50) NOT NULL DEFAULT 'scheduled'
                    CHECK (scan_type IN ('scheduled', 'manual', 'ci_trigger', 'post_deploy')),
    drift_count     INTEGER NOT NULL DEFAULT 0,
    scanned_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE drift_findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scan_id         UUID NOT NULL REFERENCES drift_scans(id) ON DELETE CASCADE,
    object_type     VARCHAR(50) NOT NULL,
    schema_name     VARCHAR(255) NOT NULL DEFAULT 'public',
    object_name     VARCHAR(255) NOT NULL,
    drift_type      VARCHAR(50) NOT NULL
                    CHECK (drift_type IN ('missing_in_db', 'missing_in_declaration',
                           'column_type_mismatch', 'constraint_mismatch',
                           'index_mismatch', 'default_mismatch', 'other')),
    expected_definition TEXT,
    actual_definition   TEXT,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('info', 'warning', 'error', 'critical')),
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_scans_env ON drift_scans(environment_id);
CREATE INDEX idx_drift_scans_status ON drift_scans(status);
CREATE INDEX idx_drift_findings_scan ON drift_findings(scan_id);
CREATE INDEX idx_drift_findings_object ON drift_findings(object_type, object_name);
```

---

## Impact Analysis

```sql
-- ============================================================
-- IMPACT ANALYSIS
-- ============================================================

CREATE TABLE impact_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_id    UUID NOT NULL REFERENCES migrations(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'completed', 'failed')),
    analyzed_by     VARCHAR(50) NOT NULL DEFAULT 'ai'
                    CHECK (analyzed_by IN ('ai', 'static_analysis', 'manual')),
    risk_score      DECIMAL(3,2),                 -- 0.00 to 1.00
    summary         TEXT,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE impact_findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID NOT NULL REFERENCES impact_analyses(id) ON DELETE CASCADE,
    finding_type    VARCHAR(50) NOT NULL
                    CHECK (finding_type IN ('orm_model_reference', 'query_reference',
                           'api_endpoint_reference', 'report_reference',
                           'downstream_migration_conflict', 'data_loss_risk',
                           'performance_impact', 'lock_risk')),
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('info', 'warning', 'error', 'critical')),
    affected_file   TEXT,                         -- source file path
    affected_line   INTEGER,                      -- line number in source
    description     TEXT NOT NULL,
    recommendation  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_impact_analyses_migration ON impact_analyses(migration_id);
CREATE INDEX idx_impact_findings_analysis ON impact_findings(analysis_id);
CREATE INDEX idx_impact_findings_type ON impact_findings(finding_type);
```

---

## Audit Logging

```sql
-- ============================================================
-- AUDIT LOGGING (ISO 27001 / SOC 2 Compliance)
-- ============================================================

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    project_id      UUID REFERENCES projects(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,         -- e.g., 'migration.created', 'approval.approved'
    resource_type   VARCHAR(100) NOT NULL,         -- e.g., 'migration', 'environment', 'user'
    resource_id     UUID NOT NULL,
    ip_address      INET,
    user_agent      TEXT,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition by month for efficient querying and retention management
-- CREATE TABLE audit_logs_2026_05 PARTITION OF audit_logs
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_logs_org ON audit_logs(organization_id);
CREATE INDEX idx_audit_logs_project ON audit_logs(project_id);
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_created ON audit_logs(created_at);
```

---

## Notifications

```sql
-- ============================================================
-- NOTIFICATIONS
-- ============================================================

CREATE TABLE notification_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    channel_type    VARCHAR(50) NOT NULL
                    CHECK (channel_type IN ('slack', 'email', 'webhook', 'teams')),
    config          JSONB NOT NULL,                -- channel-specific config
    -- Example: {"webhook_url": "https://hooks.slack.com/...", "channel": "#db-migrations"}
    events          TEXT[] NOT NULL DEFAULT '{}',   -- events to subscribe to
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      UUID NOT NULL REFERENCES notification_channels(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'sent', 'failed')),
    sent_at         TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_channel ON notifications(channel_id);
CREATE INDEX idx_notifications_status ON notifications(status);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Access Management | 4 | organizations, users, org_memberships, api_tokens |
| Project & Environment Management | 4 | projects, project_memberships, environments, database_connections |
| Schema Declarations | 2 | schema_declarations, schema_objects |
| Migration Management | 3 | migrations, migration_dependencies, migration_affected_objects |
| Migration Execution | 2 | migration_executions, migration_locks |
| Approval Workflows | 3 | approval_policies, approval_requests, approval_decisions |
| Drift Detection | 2 | drift_scans, drift_findings |
| Impact Analysis | 2 | impact_analyses, impact_findings |
| Audit Logging | 1 | audit_logs (partitioned by month) |
| Notifications | 2 | notification_channels, notifications |
| **Total** | **25** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation across services and environments without coordination, which is essential for a multi-tenant platform that may run in multiple regions.

2. **Separate `schema_declarations` and `migrations` tables** — the declarative desired-state model (Atlas-style) and the imperative migration scripts (Flyway-style) coexist. A migration can be generated from a declaration diff or authored manually.

3. **`migration_affected_objects` as an explicit junction table** — rather than parsing SQL at query time, the affected schema objects are pre-computed and stored. This enables fast "what migrations touch table X?" queries without full-text search.

4. **`migration_locks` table following Liquibase's DATABASECHANGELOGLOCK pattern** — prevents concurrent migration execution against the same environment, which is critical for production safety.

5. **Approval policies are environment-scoped with project fallback** — `environment_id` is nullable; NULL means the policy applies project-wide. This allows "production requires 2 DBA approvals" while development auto-approves.

6. **Drift findings are point-in-time snapshots** — each `drift_scan` captures a full snapshot of divergence. Historical scans are preserved (not overwritten) to track drift trends over time.

7. **Audit logs are append-only and partitioned** — ISO 27001 and SOC 2 require tamper-evident logs. The `audit_logs` table is designed for monthly partitioning, enabling efficient retention policies and time-range queries.

8. **Credentials are never stored in the database** — `database_connections.secret_ref` points to an external secret manager (Vault, AWS Secrets Manager). Only the reference is stored.

9. **Risk level is computed and stored on the migration** — rather than computing risk at query time, `migrations.risk_level` is set during migration analysis and used for approval routing.

10. **DORA metrics are derivable from execution data** — `migration_executions.execution_time_ms`, `status`, and `executed_at` provide the raw data for change failure rate and mean time to recovery calculations.
