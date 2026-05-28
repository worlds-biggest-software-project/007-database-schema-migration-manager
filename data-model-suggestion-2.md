# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Database Schema Migration Manager · Created: 2026-05-11

## Philosophy

This model treats every state change in the migration management platform as an immutable event. The event store is the single source of truth; all current-state views (projects, migrations, environments) are materialized projections rebuilt from the event stream. This is the CQRS (Command Query Responsibility Segregation) pattern applied to migration management itself.

The approach is deeply aligned with the project's core value proposition: a migration manager that provides full audit trails, drift root-cause analysis, and the ability to answer temporal queries ("what was the declared schema on March 15th?"). Event sourcing is the industry standard for implementing audit trails (as documented by Microsoft Azure Architecture Center and AWS Prescriptive Guidance), and it naturally supports the SOC 2 / ISO 27001 compliance requirements that enterprise buyers demand.

The PostgreSQL event store pattern uses an `events` table with `aggregate_id`, `event_type`, `payload` (JSONB), and a `sequence` number, with an optional `snapshots` table for performance. This model extends that pattern with domain-specific aggregate types and read model tables optimized for the migration management query patterns.

**Best for:** Teams building a platform where full auditability is non-negotiable, temporal queries are a core feature ("show me the state of this project's schema on any date"), and the platform needs to support AI-powered root-cause analysis by replaying event sequences.

**Trade-offs:**
- (+) Complete, immutable audit trail — every state change is preserved forever
- (+) Temporal queries are trivial: replay events to any point in time
- (+) AI drift root-cause analysis can replay events to correlate drift with deployments
- (+) Natural fit for SOC 2 / ISO 27001 compliance — tamper-evident by design
- (+) Event replay enables powerful analytics and machine learning on change patterns
- (-) Higher implementation complexity — developers must think in events, not CRUD
- (-) Read performance depends on materialized views being kept up to date
- (-) Event schema evolution is challenging — events are immutable, so new event versions must be forward-compatible
- (-) Debugging requires understanding both the event stream and the projections
- (-) Storage grows unboundedly; requires snapshot strategy for long-lived aggregates

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 27001 | The event store IS the audit trail — immutable by design, not bolted on |
| SOC 2 Type II | Tamper-evident event log with cryptographic chaining satisfies control requirements |
| ISO/IEC 25012 | Data consistency enforced via event ordering and aggregate invariants |
| OpenGitOps v1.0 | Desired-state declarations are events; reconciliation compares projected state vs. live DB |
| DORA Metrics | Event timestamps enable precise measurement of change lead time, failure rate, and MTTR |
| Semantic Versioning 2.0.0 | Event types are versioned (e.g., `MigrationCreated.v2`) for forward compatibility |
| Apache Avro | Event payloads can adopt Avro schema evolution for backward/forward compatibility |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- EVENT STORE — THE SINGLE SOURCE OF TRUTH
-- ============================================================

-- Core event store table
-- Follows the PostgreSQL event sourcing pattern:
-- aggregate_type + aggregate_id + sequence = unique event identity
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,
    -- Aggregate types: 'organization', 'project', 'environment',
    -- 'migration', 'approval_request', 'drift_scan', 'user'
    aggregate_id    UUID NOT NULL,
    sequence        BIGINT NOT NULL,               -- monotonically increasing per aggregate
    event_type      VARCHAR(200) NOT NULL,
    -- Event naming convention: <Aggregate><PastTenseVerb>[.v<N>]
    -- Examples: 'MigrationCreated.v1', 'MigrationApproved.v1',
    --           'MigrationExecuted.v1', 'DriftDetected.v1',
    --           'SchemaDeclarationPublished.v1'
    event_version   INTEGER NOT NULL DEFAULT 1,
    payload         JSONB NOT NULL,                -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "user_id": "uuid",
    --   "ip_address": "1.2.3.4",
    --   "user_agent": "...",
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid"       -- the event that caused this event
    -- }
    causation_id    UUID,                          -- event that directly caused this event
    correlation_id  UUID,                          -- shared ID across a chain of related events
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, sequence)
);

-- Optimistic concurrency control: ensure no two events have the same
-- sequence for the same aggregate
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_correlation ON events(correlation_id);

-- GIN index on payload for flexible querying across event types
CREATE INDEX idx_events_payload ON events USING GIN (payload jsonb_path_ops);

-- ============================================================
-- SNAPSHOTS — Performance optimization for long-lived aggregates
-- ============================================================

CREATE TABLE snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    snapshot_data   JSONB NOT NULL,                -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, last_sequence)
);

CREATE INDEX idx_snapshots_aggregate ON snapshots(aggregate_type, aggregate_id, last_sequence DESC);

-- ============================================================
-- EVENT SUBSCRIPTIONS — For projection rebuilding and notifications
-- ============================================================

CREATE TABLE event_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_name VARCHAR(200) NOT NULL UNIQUE,
    -- e.g., 'migration_read_model', 'audit_log_projection', 'slack_notifier'
    last_processed_id UUID,                        -- last event.id processed
    last_processed_at TIMESTAMPTZ,
    status          VARCHAR(50) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'paused', 'rebuilding', 'error')),
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Type Catalog

```sql
-- ============================================================
-- EVENT TYPE EXAMPLES (documented as comments, not a table)
-- ============================================================

-- Organization Events:
-- 'OrganizationCreated.v1'    payload: {"name": "...", "slug": "...", "plan_tier": "free"}
-- 'OrganizationUpdated.v1'    payload: {"changes": {"name": "new_name"}}
-- 'MemberInvited.v1'          payload: {"user_id": "...", "role": "developer"}
-- 'MemberRoleChanged.v1'      payload: {"user_id": "...", "old_role": "developer", "new_role": "dba"}
-- 'MemberRemoved.v1'          payload: {"user_id": "..."}

-- Project Events:
-- 'ProjectCreated.v1'         payload: {"name": "...", "slug": "...", "org_id": "..."}
-- 'ProjectArchived.v1'        payload: {"reason": "..."}
-- 'VcsConnected.v1'           payload: {"repo_url": "...", "provider": "github"}

-- Environment Events:
-- 'EnvironmentCreated.v1'     payload: {"name": "production", "tier": "production", "promotion_order": 2}
-- 'ConnectionConfigured.v1'   payload: {"engine": "postgresql", "host": "...", "database": "...", "secret_ref": "vault://..."}
-- 'ApprovalPolicySet.v1'      payload: {"min_approvals": 2, "required_roles": ["dba"]}

-- Schema Declaration Events:
-- 'SchemaDeclarationPublished.v1'  payload: {"version": "1.0.0", "format": "sql", "content_hash": "...", "content": "..."}
-- 'SchemaObjectDiscovered.v1'      payload: {"object_type": "table", "name": "users", "definition": "..."}

-- Migration Events:
-- 'MigrationCreated.v1'       payload: {"version": "V001", "description": "...", "sql_up": "...", "sql_down": "...", "checksum": "...", "source": "declarative_diff"}
-- 'MigrationRiskAssessed.v1'  payload: {"risk_level": "high", "is_destructive": true, "risk_factors": [...]}
-- 'MigrationImpactAnalyzed.v1' payload: {"risk_score": 0.75, "findings": [...]}
-- 'MigrationApprovalRequested.v1' payload: {"environment_id": "...", "policy_id": "..."}
-- 'MigrationApproved.v1'      payload: {"reviewer_id": "...", "comment": "..."}
-- 'MigrationRejected.v1'      payload: {"reviewer_id": "...", "comment": "...", "reason": "data_loss_risk"}
-- 'MigrationExecutionStarted.v1'  payload: {"environment_id": "...", "connection_id": "..."}
-- 'MigrationExecutionSucceeded.v1' payload: {"execution_time_ms": 1234}
-- 'MigrationExecutionFailed.v1'    payload: {"error_message": "...", "error_detail": "..."}
-- 'MigrationRolledBack.v1'    payload: {"rollback_sql": "...", "reason": "..."}

-- Drift Events:
-- 'DriftScanStarted.v1'       payload: {"environment_id": "...", "scan_type": "scheduled"}
-- 'DriftDetected.v1'          payload: {"object_type": "column", "object_name": "users.email", "drift_type": "column_type_mismatch", "expected": "varchar(255)", "actual": "text"}
-- 'DriftResolved.v1'          payload: {"finding_id": "...", "resolution": "applied_migration"}
-- 'DriftScanCompleted.v1'     payload: {"drift_count": 3, "status": "drifted"}

-- Conflict Events:
-- 'ConflictDetected.v1'       payload: {"migration_a_id": "...", "migration_b_id": "...", "conflict_type": "column_rename", "description": "..."}
-- 'ConflictResolved.v1'       payload: {"resolution": "merge", "resolved_migration_id": "..."}
```

---

## Read Models (Query Side — Materialized Projections)

```sql
-- ============================================================
-- READ MODELS — Materialized from event stream
-- These tables are fully rebuildable by replaying events.
-- They exist for query performance only.
-- ============================================================

-- Current state of organizations (projected from Organization* events)
CREATE TABLE rm_organizations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL,
    member_count    INTEGER NOT NULL DEFAULT 0,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    -- projection metadata
    last_event_id   UUID NOT NULL,
    last_event_seq  BIGINT NOT NULL
);

-- Current state of projects
CREATE TABLE rm_projects (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    vcs_repo_url    TEXT,
    vcs_provider    VARCHAR(50),
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    migration_count INTEGER NOT NULL DEFAULT 0,
    last_migration_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_seq  BIGINT NOT NULL
);

-- Current state of environments
CREATE TABLE rm_environments (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    name            VARCHAR(100) NOT NULL,
    tier            VARCHAR(50) NOT NULL,
    promotion_order INTEGER NOT NULL,
    engine          VARCHAR(50),
    current_version VARCHAR(50),                   -- last applied migration version
    last_drift_status VARCHAR(50),
    last_drift_scan_at TIMESTAMPTZ,
    requires_approval BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_seq  BIGINT NOT NULL
);

-- Current state of migrations (denormalized for fast listing)
CREATE TABLE rm_migrations (
    id              UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    version         VARCHAR(50) NOT NULL,
    description     VARCHAR(500) NOT NULL,
    migration_type  VARCHAR(50) NOT NULL,
    source          VARCHAR(50) NOT NULL,
    risk_level      VARCHAR(20) NOT NULL,
    is_destructive  BOOLEAN NOT NULL,
    checksum        VARCHAR(64) NOT NULL,
    authored_by_id  UUID NOT NULL,
    authored_by_name VARCHAR(255),                 -- denormalized for display
    -- Execution status per environment (denormalized)
    dev_status      VARCHAR(50),
    staging_status  VARCHAR(50),
    prod_status     VARCHAR(50),
    -- Approval status
    approval_status VARCHAR(50),
    -- Impact analysis
    risk_score      DECIMAL(3,2),
    impact_finding_count INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_seq  BIGINT NOT NULL
);

-- Drift findings (current unresolved drift across all environments)
CREATE TABLE rm_drift_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    environment_id  UUID NOT NULL,
    project_id      UUID NOT NULL,
    object_type     VARCHAR(50) NOT NULL,
    schema_name     VARCHAR(255) NOT NULL,
    object_name     VARCHAR(255) NOT NULL,
    drift_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    first_detected_at TIMESTAMPTZ NOT NULL,
    last_detected_at  TIMESTAMPTZ NOT NULL,
    detection_count INTEGER NOT NULL DEFAULT 1,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    last_event_id   UUID NOT NULL,
    last_event_seq  BIGINT NOT NULL
);

-- DORA metrics (pre-computed from execution events)
CREATE TABLE rm_dora_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL,
    environment_id  UUID NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    deployment_frequency INTEGER NOT NULL DEFAULT 0,
    lead_time_avg_ms    BIGINT,
    change_failure_rate DECIMAL(5,4),              -- 0.0000 to 1.0000
    mttr_avg_ms         BIGINT,                    -- mean time to recovery
    last_event_id   UUID NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    UNIQUE (project_id, environment_id, period_start)
);

CREATE INDEX idx_rm_projects_org ON rm_projects(organization_id);
CREATE INDEX idx_rm_environments_project ON rm_environments(project_id);
CREATE INDEX idx_rm_migrations_project ON rm_migrations(project_id);
CREATE INDEX idx_rm_drift_env ON rm_drift_summary(environment_id) WHERE NOT is_resolved;
CREATE INDEX idx_rm_dora_project ON rm_dora_metrics(project_id, period_start);
```

---

## Temporal Queries (Event Replay Examples)

```sql
-- ============================================================
-- EXAMPLE: What was the state of project X on March 15, 2026?
-- Replay all events for the project aggregate up to that date.
-- ============================================================

-- Get all events for a project up to a specific date
SELECT event_type, payload, created_at
FROM events
WHERE aggregate_type = 'project'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY sequence ASC;

-- ============================================================
-- EXAMPLE: Find the root cause of drift on production
-- Correlate DriftDetected events with MigrationExecuted and
-- deployment events in the same time window.
-- ============================================================

WITH drift_event AS (
    SELECT correlation_id, created_at, payload
    FROM events
    WHERE event_type = 'DriftDetected.v1'
      AND aggregate_id = :environment_id
    ORDER BY created_at DESC
    LIMIT 1
),
related_events AS (
    SELECT e.event_type, e.payload, e.created_at, e.aggregate_type
    FROM events e, drift_event d
    WHERE e.created_at BETWEEN d.created_at - INTERVAL '24 hours' AND d.created_at
      AND e.aggregate_type IN ('migration', 'environment')
    ORDER BY e.created_at ASC
)
SELECT * FROM related_events;

-- ============================================================
-- EXAMPLE: Rebuild a read model from scratch
-- (Used during deployment or after a projection bug fix)
-- ============================================================

-- Step 1: Truncate the read model
-- TRUNCATE rm_migrations;

-- Step 2: Replay all migration events in order
SELECT id, aggregate_id, event_type, payload, sequence
FROM events
WHERE aggregate_type = 'migration'
ORDER BY aggregate_id, sequence ASC;
-- Application code processes each event and INSERTs/UPDATEs rm_migrations
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write) | 3 | events, snapshots, event_subscriptions |
| Read Models (Query) | 6 | rm_organizations, rm_projects, rm_environments, rm_migrations, rm_drift_summary, rm_dora_metrics |
| **Total** | **9** | Read models are fully rebuildable from events |

---

## Key Design Decisions

1. **Single `events` table with `aggregate_type` discriminator** — rather than separate event tables per aggregate (e.g., `migration_events`, `project_events`), a single table simplifies cross-aggregate correlation queries and event subscription management. The `aggregate_type` + `aggregate_id` + `sequence` composite unique constraint provides the event stream identity.

2. **JSONB payload with GIN index** — event payloads are stored as JSONB for flexibility (each event type has different fields) while maintaining full query capability. GIN indexes with `jsonb_path_ops` enable fast containment queries across the event store.

3. **Causation and correlation IDs for event chain tracking** — `causation_id` links an event to its direct cause; `correlation_id` links all events in a chain (e.g., a migration approval → execution → completion chain shares a correlation ID). This is essential for AI-powered root-cause analysis.

4. **Read models are prefixed `rm_`** — clear naming convention distinguishes materialized projections from the source-of-truth event store. Any `rm_` table can be dropped and rebuilt from events without data loss.

5. **Event type versioning in the name** — `MigrationCreated.v1` allows evolution of event schemas. When the payload structure changes, a new version (`MigrationCreated.v2`) is introduced. Projection code handles all versions. This follows the Apache Avro schema evolution philosophy.

6. **Snapshots for long-lived aggregates** — projects with thousands of migrations would be expensive to replay from event zero. Periodic snapshots store the computed aggregate state at a specific sequence number; replay continues from the snapshot forward.

7. **`event_subscriptions` table for projection tracking** — each read model projection tracks which event it last processed. This enables catch-up after downtime, parallel projection rebuilding, and monitoring projection lag.

8. **DORA metrics as a first-class read model** — rather than computing DORA metrics on the fly, `rm_dora_metrics` is a pre-computed projection updated by migration execution events. This aligns with the DORA framework's emphasis on measurable delivery performance.

9. **No delete operations anywhere** — events are immutable. "Deletion" is modeled as an event (e.g., `ProjectArchived.v1`). Read models may remove rows, but the event store preserves the full history. This is the core SOC 2 / ISO 27001 value proposition.

10. **Partition-ready events table** — the `events` table can be partitioned by `created_at` (range partitioning by month/year) for performance with billions of events. PostgreSQL's partition pruning automatically excludes irrelevant partitions during temporal queries.
