# Data Model Suggestion 4: Graph-Relational (Schema Dependency Graph)

> Project: Database Schema Migration Manager · Created: 2026-05-11

## Philosophy

This model places a property graph at the center of the architecture: every schema object (table, column, index, constraint, function, view) and every code artifact (ORM model, query, API endpoint) is a node in a directed graph, with edges representing dependencies, impacts, and ownership relationships. The operational CRUD data (projects, environments, users) lives in conventional relational tables, but the schema intelligence — the part that enables conflict detection, impact analysis, and drift root-cause correlation — is powered by graph traversal.

The core insight is that database schema migration is fundamentally a graph problem. A column depends on a table. An index depends on columns. A foreign key connects two tables. An ORM model references columns. A query traverses tables through joins. A migration creates, alters, or destroys nodes and edges. Conflict detection is graph intersection. Impact analysis is graph reachability. Drift root-cause analysis is graph provenance tracing. None of these operations map naturally to relational JOINs — they are graph traversals of variable depth.

This model uses PostgreSQL's `ltree` extension for hierarchical paths and a pair of generic `graph_nodes` / `graph_edges` tables that implement a property graph within PostgreSQL. This avoids requiring a separate graph database (Neo4j, Neptune) while providing the graph traversal capabilities that make AI-powered impact analysis and conflict resolution tractable.

**Best for:** Teams that prioritize the AI-powered features (conflict detection, impact analysis, drift root-cause analysis) as their primary differentiator; platforms where understanding the dependency graph between schema objects, code artifacts, and migration operations is the core value proposition.

**Trade-offs:**
- (+) Graph traversal enables multi-hop queries that are impractical in relational models ("what code is affected if I drop this column?")
- (+) Conflict detection reduces to graph intersection — computationally efficient
- (+) Impact analysis is graph reachability — follow edges from a changed node to all dependents
- (+) Drift root-cause analysis can trace provenance through the graph
- (+) Natural representation of schema relationships (FK dependencies, index-column relationships)
- (-) Higher conceptual complexity — developers must understand both relational and graph paradigms
- (-) Graph queries (recursive CTEs, ltree operations) require specialized SQL knowledge
- (-) Property graph in PostgreSQL lacks the query ergonomics of a native graph database (Cypher, Gremlin)
- (-) Graph maintenance overhead — every schema change must update nodes and edges
- (-) Testing requires graph fixtures, which are harder to set up than relational test data

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 27001 | Audit trail on graph mutations; provenance edges trace who changed what and when |
| ISO/IEC 25012 | Data quality enforced via graph consistency checks (orphaned nodes, broken edges) |
| OpenGitOps v1.0 | Declared schema is a graph of desired nodes/edges; drift = difference between desired and actual graphs |
| DORA Metrics | Migration execution timing on relational tables; graph enables root-cause analysis of failures |
| SQL/PGQ (ISO 9075-16) | Anticipated alignment with SQL property graph query standard (ISO working draft) |
| JSON Schema (IETF) | `properties` JSONB on nodes/edges validated by JSON Schema per node type |

---

## Operational Tables (Relational)

```sql
-- ============================================================
-- OPERATIONAL TABLES (standard relational CRUD)
-- These tables handle the day-to-day operations of the platform.
-- The graph layer (below) handles schema intelligence.
-- ============================================================

CREATE EXTENSION IF NOT EXISTS ltree;     -- hierarchical path queries
CREATE EXTENSION IF NOT EXISTS pg_trgm;   -- trigram similarity for fuzzy search

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'developer'
                    CHECK (role IN ('owner', 'admin', 'dba', 'developer', 'viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    vcs_repo_url    TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE environments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    tier            VARCHAR(50) NOT NULL DEFAULT 'development'
                    CHECK (tier IN ('development', 'staging', 'production')),
    engine          VARCHAR(50) NOT NULL,
    connection_config JSONB NOT NULL DEFAULT '{}',
    promotion_order INTEGER NOT NULL DEFAULT 0,
    requires_approval BOOLEAN NOT NULL DEFAULT false,
    current_version VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name)
);

CREATE TABLE migrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,
    description     VARCHAR(500) NOT NULL,
    migration_type  VARCHAR(50) NOT NULL DEFAULT 'versioned',
    source          VARCHAR(50) NOT NULL DEFAULT 'manual',
    sql_up          TEXT NOT NULL,
    sql_down        TEXT,
    checksum        VARCHAR(64) NOT NULL,
    risk_level      VARCHAR(20) NOT NULL DEFAULT 'low',
    is_destructive  BOOLEAN NOT NULL DEFAULT false,
    authored_by     UUID NOT NULL REFERENCES users(id),
    commit_sha      VARCHAR(40),
    -- Link to the graph: which nodes does this migration create/modify/destroy?
    graph_snapshot_id UUID,                        -- references graph_snapshots
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, version)
);

CREATE TABLE migration_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_id    UUID NOT NULL REFERENCES migrations(id),
    environment_id  UUID NOT NULL REFERENCES environments(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'awaiting_approval', 'approved',
                           'running', 'success', 'failed', 'rolled_back')),
    executed_sql    TEXT,
    execution_time_ms INTEGER,
    executed_by     UUID REFERENCES users(id),
    executed_at     TIMESTAMPTZ,
    error_message   TEXT,
    approval_details JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_environments_project ON environments(project_id);
CREATE INDEX idx_migrations_project ON migrations(project_id);
CREATE INDEX idx_executions_migration ON migration_executions(migration_id);
CREATE INDEX idx_executions_env ON migration_executions(environment_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_created ON audit_logs(created_at);
```

---

## Graph Layer (Schema Intelligence)

```sql
-- ============================================================
-- GRAPH LAYER — Property graph for schema intelligence
-- ============================================================

-- Every schema object, code artifact, and migration is a node
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID REFERENCES environments(id),  -- NULL = project-level (declared)

    -- Node classification
    node_type       VARCHAR(50) NOT NULL,
    -- Schema nodes:    'table', 'column', 'index', 'constraint', 'trigger',
    --                  'function', 'view', 'sequence', 'enum', 'schema'
    -- Code nodes:      'orm_model', 'orm_field', 'query', 'api_endpoint',
    --                  'report', 'migration'
    -- External nodes:  'external_service', 'data_pipeline'

    -- Hierarchical path using ltree for efficient ancestor/descendant queries
    -- Format: schema.table.column or src.models.user.email_field
    path            ltree NOT NULL,
    -- Examples:
    --   'public.users'                    (table)
    --   'public.users.email'              (column)
    --   'public.users.idx_users_email'    (index)
    --   'src.models.User'                 (ORM model)
    --   'src.models.User.email'           (ORM field)
    --   'src.api.get_users'               (API endpoint)

    -- Human-readable identifiers
    qualified_name  VARCHAR(500) NOT NULL,         -- e.g., 'public.users.email'
    display_name    VARCHAR(255) NOT NULL,         -- e.g., 'email'

    -- Node properties (type-specific attributes)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Column node example:
    -- {
    --   "data_type": "varchar(320)",
    --   "nullable": false,
    --   "default": null,
    --   "is_primary_key": false,
    --   "is_unique": true,
    --   "comment": "User email address — RFC 5321 compliant"
    -- }
    --
    -- ORM model node example:
    -- {
    --   "file_path": "src/models/user.ts",
    --   "line_number": 15,
    --   "framework": "prisma",
    --   "model_name": "User",
    --   "language": "typescript"
    -- }
    --
    -- Index node example:
    -- {
    --   "index_type": "btree",
    --   "is_unique": true,
    --   "columns": ["email"],
    --   "is_concurrent": false
    -- }

    -- State tracking
    status          VARCHAR(50) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'deprecated', 'deleted', 'proposed')),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Directed edges between nodes with typed relationships
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,

    -- Relationship classification
    edge_type       VARCHAR(50) NOT NULL,
    -- Schema relationships:
    --   'belongs_to'       column → table, index → table
    --   'references'       FK column → referenced table
    --   'depends_on'       view → table, function → table
    --   'constrains'       constraint → column(s)
    --   'indexes'          index → column(s)
    --
    -- Code-to-schema relationships:
    --   'maps_to'          ORM field → column
    --   'queries'          query/endpoint → table
    --   'reads_from'       query → column (SELECT)
    --   'writes_to'        query → column (INSERT/UPDATE)
    --   'joins_on'         query → FK relationship
    --
    -- Migration relationships:
    --   'creates'          migration → node (new object)
    --   'alters'           migration → node (modified object)
    --   'drops'            migration → node (removed object)
    --   'renames'          migration → node (renamed object, properties has old_name)
    --
    -- Conflict/dependency relationships:
    --   'conflicts_with'   migration → migration
    --   'supersedes'       migration → migration
    --   'caused_drift'     event → drift finding

    -- Edge properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- FK reference example:
    -- {
    --   "constraint_name": "fk_orders_user_id",
    --   "on_delete": "CASCADE",
    --   "on_update": "NO ACTION"
    -- }
    --
    -- ORM mapping example:
    -- {
    --   "field_name": "email",
    --   "mapped_column": "email",
    --   "is_relation": false
    -- }
    --
    -- Migration operation example:
    -- {
    --   "operation": "alter_column",
    --   "old_type": "varchar(255)",
    --   "new_type": "varchar(320)"
    -- }

    weight          DECIMAL(5,2) DEFAULT 1.0,      -- for weighted graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_node_id, target_node_id, edge_type)
);

-- Graph snapshots: point-in-time captures of the graph state
-- Used for temporal comparisons and drift detection
CREATE TABLE graph_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    environment_id  UUID REFERENCES environments(id),
    snapshot_type   VARCHAR(50) NOT NULL
                    CHECK (snapshot_type IN ('declared', 'actual', 'pre_migration', 'post_migration')),
    node_count      INTEGER NOT NULL DEFAULT 0,
    edge_count      INTEGER NOT NULL DEFAULT 0,
    content_hash    VARCHAR(64) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Snapshot membership: which nodes/edges belong to a snapshot
CREATE TABLE graph_snapshot_members (
    snapshot_id     UUID NOT NULL REFERENCES graph_snapshots(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    PRIMARY KEY (snapshot_id, node_id)
);

-- ============================================================
-- GRAPH INDEXES — Critical for traversal performance
-- ============================================================

-- ltree indexes for hierarchical queries
CREATE INDEX idx_nodes_path_gist ON graph_nodes USING GIST (path);
CREATE INDEX idx_nodes_path_btree ON graph_nodes USING BTREE (path);

-- Standard indexes for graph traversal
CREATE INDEX idx_nodes_project ON graph_nodes(project_id);
CREATE INDEX idx_nodes_env ON graph_nodes(environment_id);
CREATE INDEX idx_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_nodes_qualified ON graph_nodes(qualified_name);
CREATE INDEX idx_nodes_status ON graph_nodes(status) WHERE status = 'active';

CREATE INDEX idx_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_edges_project ON graph_edges(project_id);

-- GIN indexes for JSONB property queries
CREATE INDEX idx_nodes_properties ON graph_nodes USING GIN (properties jsonb_path_ops);
CREATE INDEX idx_edges_properties ON graph_edges USING GIN (properties jsonb_path_ops);

-- Trigram index for fuzzy name search
CREATE INDEX idx_nodes_name_trgm ON graph_nodes USING GIN (qualified_name gin_trgm_ops);

CREATE INDEX idx_snapshot_members ON graph_snapshot_members(node_id);
```

---

## Graph Query Examples

```sql
-- ============================================================
-- IMPACT ANALYSIS: What code is affected if I drop column "users.email"?
-- ============================================================

-- Step 1: Find the column node
WITH target_column AS (
    SELECT id FROM graph_nodes
    WHERE project_id = :project_id
      AND node_type = 'column'
      AND qualified_name = 'public.users.email'
      AND status = 'active'
),

-- Step 2: Recursive traversal — find all nodes reachable via
-- 'maps_to', 'reads_from', 'writes_to', 'queries', 'depends_on' edges
impacted_nodes AS (
    -- Base case: direct dependents of the column
    SELECT
        gn.id, gn.node_type, gn.qualified_name, gn.properties,
        1 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    JOIN target_column tc ON ge.target_node_id = tc.id
    WHERE ge.edge_type IN ('maps_to', 'reads_from', 'writes_to',
                           'queries', 'depends_on', 'indexes', 'constrains')

    UNION ALL

    -- Recursive case: dependents of dependents
    SELECT
        gn.id, gn.node_type, gn.qualified_name, gn.properties,
        prev.depth + 1,
        prev.path || gn.id
    FROM impacted_nodes prev
    JOIN graph_edges ge ON ge.target_node_id = prev.id
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    WHERE prev.depth < 5                           -- max traversal depth
      AND NOT gn.id = ANY(prev.path)               -- prevent cycles
      AND ge.edge_type IN ('maps_to', 'reads_from', 'writes_to',
                           'queries', 'depends_on')
)
SELECT node_type, qualified_name, properties, depth
FROM impacted_nodes
ORDER BY depth, node_type;

-- ============================================================
-- CONFLICT DETECTION: Do two migrations touch the same objects?
-- ============================================================

WITH migration_a_targets AS (
    SELECT ge.target_node_id
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    WHERE gn.node_type = 'migration'
      AND gn.properties->>'migration_id' = :migration_a_id
      AND ge.edge_type IN ('creates', 'alters', 'drops', 'renames')
),
migration_b_targets AS (
    SELECT ge.target_node_id
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    WHERE gn.node_type = 'migration'
      AND gn.properties->>'migration_id' = :migration_b_id
      AND ge.edge_type IN ('creates', 'alters', 'drops', 'renames')
),
conflicts AS (
    SELECT a.target_node_id
    FROM migration_a_targets a
    INNER JOIN migration_b_targets b ON a.target_node_id = b.target_node_id
)
SELECT gn.node_type, gn.qualified_name, gn.properties
FROM conflicts c
JOIN graph_nodes gn ON gn.id = c.target_node_id;

-- ============================================================
-- HIERARCHICAL QUERY: All columns under the "users" table
-- Uses ltree for efficient subtree queries
-- ============================================================

SELECT id, node_type, qualified_name, properties
FROM graph_nodes
WHERE path <@ 'public.users'::ltree               -- all descendants of public.users
  AND status = 'active'
ORDER BY path;

-- ============================================================
-- DRIFT ROOT-CAUSE: Trace provenance of a drifted column
-- Find all migrations and manual changes that touched this node
-- ============================================================

SELECT
    ge.edge_type AS operation,
    gn_source.node_type AS actor_type,
    gn_source.qualified_name AS actor,
    gn_source.properties AS actor_props,
    ge.properties AS operation_details,
    ge.created_at
FROM graph_edges ge
JOIN graph_nodes gn_target ON gn_target.id = ge.target_node_id
JOIN graph_nodes gn_source ON gn_source.id = ge.source_node_id
WHERE gn_target.qualified_name = 'public.users.email'
  AND ge.edge_type IN ('creates', 'alters', 'drops', 'renames', 'caused_drift')
ORDER BY ge.created_at DESC;

-- ============================================================
-- FOREIGN KEY DEPENDENCY CHAIN: What tables depend on "users"?
-- ============================================================

WITH RECURSIVE fk_chain AS (
    -- Base: tables with FKs referencing "users"
    SELECT
        gn_source.id,
        gn_source.qualified_name AS dependent_table,
        ge.properties->>'constraint_name' AS fk_name,
        1 AS depth,
        ARRAY[gn_source.id] AS visited
    FROM graph_edges ge
    JOIN graph_nodes gn_target ON gn_target.id = ge.target_node_id
    JOIN graph_nodes gn_source ON gn_source.id = ge.source_node_id
    WHERE gn_target.qualified_name = 'public.users'
      AND ge.edge_type = 'references'
      AND gn_source.node_type = 'table'

    UNION ALL

    -- Recursive: tables with FKs referencing the dependent tables
    SELECT
        gn_source.id,
        gn_source.qualified_name,
        ge.properties->>'constraint_name',
        prev.depth + 1,
        prev.visited || gn_source.id
    FROM fk_chain prev
    JOIN graph_edges ge ON ge.target_node_id = prev.id
    JOIN graph_nodes gn_source ON gn_source.id = ge.source_node_id
    WHERE ge.edge_type = 'references'
      AND gn_source.node_type = 'table'
      AND NOT gn_source.id = ANY(prev.visited)
      AND prev.depth < 10
)
SELECT dependent_table, fk_name, depth
FROM fk_chain
ORDER BY depth, dependent_table;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Operational (Relational) | 7 | organizations, users, projects, environments, migrations, migration_executions, audit_logs |
| Graph Layer | 4 | graph_nodes, graph_edges, graph_snapshots, graph_snapshot_members |
| **Total** | **11** | Graph layer stores all schema intelligence |

---

## Key Design Decisions

1. **Dual-layer architecture: relational + graph** — operational CRUD (creating projects, executing migrations, managing users) uses conventional relational tables for simplicity and familiarity. Schema intelligence (impact analysis, conflict detection, drift root-cause) uses the graph layer for traversal-heavy queries. This avoids forcing all operations through a graph model while capturing the graph benefits where they matter most.

2. **PostgreSQL `ltree` for hierarchical paths** — schema objects naturally form hierarchies (`schema.table.column`). The `ltree` extension provides efficient ancestor/descendant queries (`<@`, `@>`, `~` operators) without recursive CTEs. This is critical for "find all objects under this table" queries that occur on every migration analysis.

3. **Property graph pattern (`graph_nodes` + `graph_edges`)** — rather than modeling each relationship type as a separate junction table (which would require 15+ tables for all the edge types), a generic property graph with typed edges and JSONB properties captures all relationship patterns in two tables. The `edge_type` column discriminates between relationship types.

4. **Code artifacts as graph nodes alongside schema objects** — ORM models, queries, and API endpoints are first-class nodes in the same graph as tables and columns. This enables the platform's key differentiator: "if I change this column, what code breaks?" is a single graph traversal rather than a separate code analysis system.

5. **Graph snapshots for temporal comparison** — `graph_snapshots` capture the node/edge state at specific points (before/after migration, declared vs. actual). Drift detection reduces to graph diff: compare the "declared" snapshot with the "actual" snapshot. This is more powerful than text-based schema diffing because it preserves relationships.

6. **Weighted edges for prioritized impact analysis** — the `weight` column on `graph_edges` allows the AI impact analyzer to prioritize: a `writes_to` edge (data modification) has higher weight than a `reads_from` edge (read-only query). Weighted traversal produces impact scores that reflect actual risk.

7. **Migration nodes in the graph** — each migration is represented as both a row in the `migrations` relational table (for CRUD) and a node in `graph_nodes` (for relationship tracking). The migration node has `creates`, `alters`, `drops` edges to the schema objects it affects. Conflict detection reduces to finding two migration nodes with overlapping target sets.

8. **Trigram index for fuzzy search** — `pg_trgm` enables fuzzy search on `qualified_name`, so developers can search for "usr.emal" and find "public.users.email". This is important for the natural-language migration generation feature where AI-generated references may not exactly match object names.

9. **No separate graph database** — while Neo4j or Amazon Neptune would provide native Cypher/Gremlin query ergonomics, keeping the graph in PostgreSQL eliminates an additional infrastructure dependency, simplifies transactions (graph mutations and relational updates in the same transaction), and reduces operational complexity. The recursive CTE and ltree patterns cover the required query patterns.

10. **Graph maintenance is event-driven** — when a migration is created, analyzed, or executed, the application updates the graph (adding/modifying/marking nodes and edges). This keeps the graph in sync with the operational data without requiring a separate synchronization mechanism.
