# Database Schema Migration Manager — Phased Development Plan

> Project: 007-database-schema-migration-manager · Created: 2026-05-11
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | Best balance of CLI ergonomics, ecosystem breadth for database drivers, and type safety. The primary buyer personas (platform engineers, backend devs) are heavily TypeScript-native. Python was considered but TypeScript's type system better supports the declarative schema modeling and AST manipulation required. |
| Runtime | Node.js 22 LTS | Native ESM, stable `node:test` runner, and mature database driver ecosystem (pg, mysql2, better-sqlite3). |
| Database (platform state) | PostgreSQL 16 | The platform's own state store. Chosen for JSONB capabilities, `ltree` extension, recursive CTE support, and partitioning — all required by the hybrid data model. Self-dogfooding: a migration manager should run on the strongest database it supports. |
| Data Model | Hybrid Relational + JSONB (Data Model Suggestion 3) | 11 tables vs. 25 in the normalized model. Absorbs cross-engine variability (PostgreSQL vs. MySQL vs. SQLite connection configs, DDL options) elegantly via JSONB. Faster time-to-MVP. GIN indexes provide query performance parity with normalized tables for the core access patterns. The Graph-Relational model (Suggestion 4) is adopted selectively for the impact analysis layer in Phase 7. |
| CLI Framework | `commander` + `ink` (React for CLI) | `commander` for argument parsing; `ink` for rich interactive terminal output (migration plan previews, drift reports). Matches Atlas and Flyway CLI ergonomics. |
| ORM / Query Builder | Drizzle ORM | Type-safe SQL with zero runtime overhead. Supports PostgreSQL, MySQL, SQLite — the three MVP target databases. Schema-first approach aligns with the project's declarative philosophy. |
| Schema Parsing | `pg-query-emscripten` (PostgreSQL parser) + `sql-parser-cst` | AST-based SQL parsing for migration analysis, DDL extraction, and schema diffing. `pg-query-emscripten` provides the PostgreSQL libpg_query parser compiled to WASM. `sql-parser-cst` handles MySQL and SQLite dialects. |
| Web Framework (API) | Hono | Lightweight, edge-ready HTTP framework. OpenAPI 3.1 schema generation via `@hono/zod-openapi`. Runs on Node.js and edge runtimes. |
| Web UI | React 19 + Vite + Tailwind CSS 4 | Approval workflows, drift dashboards, and migration history require a web UI. React for component model; Vite for build tooling; Tailwind for rapid UI development. |
| Authentication | OAuth 2.0 / OIDC via `arctic` library | Supports GitHub, Google, Okta, SAML providers. Aligns with OAuth 2.0 / OIDC standards from standards.md. `arctic` is lightweight and well-maintained. |
| Testing | `vitest` + `testcontainers` | `vitest` for unit/integration tests with native TypeScript support. `testcontainers` spins up real PostgreSQL, MySQL, and SQLite instances for integration tests — critical for a migration tool that must verify actual DDL execution. |
| CI/CD | GitHub Actions | Primary CI target. The CLI produces machine-readable JSON output for CI integration. GitHub Action published as a reusable action. |
| Packaging | npm (CLI) + Docker (server) | CLI distributed as `npx schemashift` or global install. Server component distributed as Docker image for self-hosted deployments. |
| AI Integration | Anthropic Claude API via `@anthropic-ai/sdk` | Impact analysis, natural-language migration generation, conflict resolution, and drift root-cause analysis. Claude's large context window supports full-schema analysis. Prompt caching reduces cost for repeated schema analysis. |
| Secret Management | External reference (`vault://`, `aws-sm://`) | Database credentials are never stored in the platform database. Connection configs store `secret_ref` strings pointing to Vault, AWS Secrets Manager, or environment variables. |

### Project Structure

```
schemashift/
├── packages/
│   ├── core/                          # Shared kernel — types, schema parsing, SQL generation
│   │   ├── src/
│   │   │   ├── types/                 # Domain types: Migration, SchemaSnapshot, Target, etc.
│   │   │   ├── parser/                # SQL AST parsing (pg-query, sql-parser-cst)
│   │   │   ├── differ/                # Schema diff engine (declared vs. actual)
│   │   │   ├── generator/             # SQL generation from diff plans
│   │   │   ├── checksum/              # SHA-256 checksum computation
│   │   │   ├── engines/               # Database engine adapters
│   │   │   │   ├── postgresql.ts
│   │   │   │   ├── mysql.ts
│   │   │   │   └── sqlite.ts
│   │   │   └── index.ts
│   │   ├── tests/
│   │   └── package.json
│   ├── cli/                           # CLI application
│   │   ├── src/
│   │   │   ├── commands/              # CLI commands: migrate, plan, drift, init, status
│   │   │   ├── config/                # Configuration loading (schemashift.toml)
│   │   │   ├── output/               # Formatters: table, json, ink components
│   │   │   └── index.ts
│   │   ├── tests/
│   │   └── package.json
│   ├── server/                        # API server + web UI backend
│   │   ├── src/
│   │   │   ├── api/                   # Hono routes
│   │   │   ├── auth/                  # OAuth 2.0 / OIDC
│   │   │   ├── db/                    # Drizzle schema + migrations (self-managed)
│   │   │   ├── services/              # Business logic
│   │   │   ├── workers/               # Background jobs (drift scanning, notifications)
│   │   │   └── index.ts
│   │   ├── tests/
│   │   └── package.json
│   ├── web/                           # React web UI
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── pages/
│   │   │   ├── hooks/
│   │   │   └── main.tsx
│   │   └── package.json
│   └── ai/                            # AI integration layer
│       ├── src/
│       │   ├── impact-analyzer.ts     # Code impact analysis
│       │   ├── migration-generator.ts # Natural-language to SQL
│       │   ├── conflict-resolver.ts   # Multi-team conflict resolution
│       │   ├── drift-analyzer.ts      # Drift root-cause analysis
│       │   └── prompts/               # Prompt templates
│       ├── tests/
│       └── package.json
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── schemashift.toml.example           # Example configuration
├── turbo.json                         # Turborepo config
├── package.json                       # Root workspace
└── tsconfig.base.json
```

---

## Phase 1: Project Scaffold and Core Types

### Purpose

Establish the monorepo structure, define all domain types as TypeScript interfaces, configure the build toolchain, and set up the testing infrastructure. This phase produces no user-facing functionality but creates the foundation every subsequent phase builds on.

### Tasks

#### 1.1 — Initialize Monorepo with Turborepo

**What**: Create the workspace structure with `packages/core`, `packages/cli`, `packages/server`, `packages/web`, and `packages/ai`.

**Design**:

Root `package.json`:
```json
{
  "name": "schemashift",
  "private": true,
  "workspaces": ["packages/*"],
  "devDependencies": {
    "turbo": "^2.5.0",
    "typescript": "^5.7.0",
    "vitest": "^3.1.0"
  }
}
```

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}
```

Shared `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

**Testing**:
- **test-scaffold-build**: Run `turbo build` from root. Expected: all packages compile with zero errors.
- **test-scaffold-typecheck**: Run `turbo typecheck`. Expected: zero type errors across all packages.

---

#### 1.2 — Define Core Domain Types

**What**: Create the TypeScript type definitions that represent every domain concept in the platform.

**Design**:

```typescript
// packages/core/src/types/engine.ts

export type DatabaseEngine = 'postgresql' | 'mysql' | 'sqlite';

export interface ConnectionConfig {
  engine: DatabaseEngine;
  host?: string;
  port?: number;
  database: string;
  schema?: string;       // default: 'public' for PostgreSQL
  secretRef?: string;    // vault://secret/db/prod
  options: Record<string, unknown>;
}

// packages/core/src/types/schema.ts

export interface SchemaObject {
  objectType: SchemaObjectType;
  schema: string;
  name: string;
  definition: string;
  attributes: Record<string, unknown>;
}

export type SchemaObjectType =
  | 'table' | 'column' | 'index' | 'constraint'
  | 'trigger' | 'function' | 'view' | 'sequence' | 'enum' | 'extension';

export interface SchemaSnapshot {
  id: string;
  targetId: string;
  snapshotType: 'declared' | 'actual' | 'diff';
  schemaData: SchemaData;
  contentHash: string;
  capturedAt: Date;
}

export interface SchemaData {
  tables: Record<string, TableDefinition>;
  enums: Record<string, EnumDefinition>;
  extensions: string[];
  functions: Record<string, FunctionDefinition>;
}

export interface TableDefinition {
  columns: Record<string, ColumnDefinition>;
  indexes: Record<string, IndexDefinition>;
  constraints: Record<string, ConstraintDefinition>;
}

export interface ColumnDefinition {
  type: string;          // e.g., 'varchar(320)', 'uuid', 'timestamptz'
  nullable: boolean;
  default?: string;
  primaryKey: boolean;
  unique: boolean;
  references?: ForeignKeyReference;
  comment?: string;
}

export interface ForeignKeyReference {
  table: string;
  column: string;
  onDelete: 'CASCADE' | 'SET NULL' | 'SET DEFAULT' | 'RESTRICT' | 'NO ACTION';
  onUpdate: 'CASCADE' | 'SET NULL' | 'SET DEFAULT' | 'RESTRICT' | 'NO ACTION';
}

export interface IndexDefinition {
  columns: string[];
  unique: boolean;
  type: 'btree' | 'hash' | 'gin' | 'gist' | 'brin';
  concurrent: boolean;
}

export interface ConstraintDefinition {
  type: 'primary_key' | 'unique' | 'check' | 'foreign_key' | 'exclude';
  columns: string[];
  expression?: string;   // for CHECK constraints
}

export interface EnumDefinition {
  values: string[];
}

export interface FunctionDefinition {
  language: string;
  returnType: string;
  arguments: string;
  body: string;
  volatility: 'VOLATILE' | 'STABLE' | 'IMMUTABLE';
}

// packages/core/src/types/migration.ts

export interface Migration {
  id: string;
  projectId: string;
  version: string;
  description: string;
  migrationType: MigrationType;
  source: MigrationSource;
  sqlUp: string;
  sqlDown: string | null;
  checksum: string;
  analysis: MigrationAnalysis;
  engineOptions: Record<string, unknown>;
  authoredBy: string;
  commitSha: string | null;
  createdAt: Date;
  updatedAt: Date;
}

export type MigrationType = 'versioned' | 'repeatable' | 'undo' | 'baseline';
export type MigrationSource = 'manual' | 'declarative_diff' | 'ai_generated' | 'import_flyway' | 'import_liquibase';

export interface MigrationAnalysis {
  riskLevel: RiskLevel;
  isDestructive: boolean;
  riskFactors: RiskFactor[];
  affectedObjects: AffectedObject[];
  impact: ImpactSummary | null;
  aiConfidence: number | null;
  analyzedAt: Date | null;
}

export type RiskLevel = 'low' | 'medium' | 'high' | 'critical';

export interface RiskFactor {
  type: string;
  object: string;
  severity: RiskLevel;
  description: string;
}

export interface AffectedObject {
  type: SchemaObjectType;
  schema: string;
  name: string;
  operation: DDLOperation;
}

export type DDLOperation =
  | 'create' | 'alter' | 'drop' | 'rename'
  | 'add_column' | 'drop_column' | 'alter_column'
  | 'add_index' | 'drop_index';

export interface ImpactSummary {
  riskScore: number;     // 0.00 to 1.00
  ormReferences: CodeReference[];
  downstreamConflicts: string[];
  recommendation: string;
}

export interface CodeReference {
  file: string;
  line: number;
  type: 'orm_field' | 'raw_query' | 'api_endpoint' | 'report';
}

// packages/core/src/types/execution.ts

export type ExecutionStatus =
  | 'pending' | 'awaiting_approval' | 'approved' | 'rejected'
  | 'running' | 'success' | 'failed' | 'rolled_back' | 'skipped';

export interface MigrationRun {
  id: string;
  migrationId: string;
  targetId: string;
  status: ExecutionStatus;
  executedSql: string | null;
  executionTimeMs: number | null;
  executedBy: string | null;
  executedAt: Date | null;
  completedAt: Date | null;
  details: MigrationRunDetails;
  createdAt: Date;
}

export interface MigrationRunDetails {
  approvals: ApprovalDecision[];
  requiredApprovals: number;
  approvalPolicy: string | null;
  error: ExecutionError | null;
  rollback: RollbackRecord | null;
}

export interface ApprovalDecision {
  reviewerId: string;
  reviewerName: string;
  decision: 'approve' | 'reject' | 'request_changes';
  comment: string | null;
  decidedAt: Date;
}

export interface ExecutionError {
  message: string;
  detail: string | null;
  sqlState: string | null;
}

export interface RollbackRecord {
  executedAt: Date;
  executedBy: string;
  sql: string;
  success: boolean;
}

// packages/core/src/types/drift.ts

export interface DriftReport {
  id: string;
  targetId: string;
  scanType: 'scheduled' | 'manual' | 'ci_trigger' | 'post_deploy';
  status: 'clean' | 'drifted' | 'error';
  findings: DriftFinding[];
  driftCount: number;
  scannedAt: Date;
  completedAt: Date | null;
}

export interface DriftFinding {
  objectType: SchemaObjectType;
  schema: string;
  table?: string;
  column?: string;
  name?: string;
  driftType: DriftType;
  expected: string | null;
  actual: string | null;
  severity: 'info' | 'warning' | 'error' | 'critical';
  isResolved: boolean;
  probableCause?: DriftCause;
}

export type DriftType =
  | 'missing_in_db' | 'missing_in_declaration'
  | 'column_type_mismatch' | 'constraint_mismatch'
  | 'index_mismatch' | 'default_mismatch' | 'other';

export interface DriftCause {
  type: 'manual_hotfix' | 'direct_ddl' | 'migration_failure' | 'unknown';
  correlatedCommit?: string;
  correlatedUser?: string;
}

// packages/core/src/types/config.ts

export interface SchemashiftConfig {
  project: ProjectConfig;
  targets: Record<string, TargetConfig>;
}

export interface ProjectConfig {
  name: string;
  migrationsDir: string;     // default: 'migrations/'
  schemaFile: string;        // default: 'schema.sql'
  namingConvention: 'timestamp' | 'sequential'; // default: 'timestamp'
}

export interface TargetConfig {
  engine: DatabaseEngine;
  connection: ConnectionConfig;
  tier: 'development' | 'staging' | 'production';
  policies?: TargetPolicies;
}

export interface TargetPolicies {
  requiresApproval: boolean;
  minApprovals: number;
  requiredRoles: string[];
  autoApproveRisk: RiskLevel[];
  maintenanceWindow?: MaintenanceWindow;
  expandContractEnabled: boolean;
}

export interface MaintenanceWindow {
  day: string;
  start: string;
  end: string;
  timezone: string;
}
```

**Testing**:
- **test-types-compile**: Import all types in a test file; verify TypeScript compilation succeeds. No runtime tests — these are pure type definitions.
- **test-types-exhaustive**: Write a type-level test (using `ts-expect-error`) verifying that invalid enum values produce type errors: `const bad: DatabaseEngine = 'oracle'` should fail.

---

#### 1.3 — Configure Vitest and Testcontainers

**What**: Set up the test infrastructure with database containers for integration testing.

**Design**:

```typescript
// packages/core/tests/setup.ts

import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { MySqlContainer, StartedMySqlContainer } from '@testcontainers/mysql';

let pgContainer: StartedPostgreSqlContainer;
let mysqlContainer: StartedMySqlContainer;

export async function setupPostgres(): Promise<string> {
  pgContainer = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('schemashift_test')
    .start();
  return pgContainer.getConnectionUri();
}

export async function setupMysql(): Promise<string> {
  mysqlContainer = await new MySqlContainer('mysql:8.4')
    .withDatabase('schemashift_test')
    .start();
  return mysqlContainer.getConnectionUri();
}

export async function teardownAll(): Promise<void> {
  await pgContainer?.stop();
  await mysqlContainer?.stop();
}
```

`vitest.config.ts` (root):
```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    testTimeout: 60_000,   // testcontainers need time to start
    hookTimeout: 120_000,
    pool: 'forks',         // isolate tests using database containers
  },
});
```

**Testing**:
- **test-container-postgres**: Start a PostgreSQL container, execute `SELECT 1`, verify result. Expected: `{ rows: [{ ?column?: 1 }] }`.
- **test-container-mysql**: Start a MySQL container, execute `SELECT 1`, verify result.
- **test-container-cleanup**: Verify containers are stopped after test suite completes; no orphaned Docker containers.

---

## Phase 2: Database Engine Adapters and Schema Introspection

### Purpose

Build the engine adapter layer that connects to PostgreSQL, MySQL, and SQLite databases, introspects their live schemas, and returns a normalized `SchemaData` representation. This is the "actual state" half of the declarative diff equation.

### Tasks

#### 2.1 — Define the Engine Adapter Interface

**What**: Create the abstract interface that all database engine adapters implement.

**Design**:

```typescript
// packages/core/src/engines/adapter.ts

import type { ConnectionConfig, SchemaData, DatabaseEngine } from '../types';

export interface EngineAdapter {
  readonly engine: DatabaseEngine;

  connect(config: ConnectionConfig): Promise<void>;
  disconnect(): Promise<void>;
  ping(): Promise<boolean>;

  /** Introspect the live database and return a SchemaData snapshot */
  introspect(): Promise<SchemaData>;

  /** Execute raw SQL (for migration application) */
  execute(sql: string): Promise<ExecuteResult>;

  /** Execute SQL within a transaction */
  executeInTransaction(statements: string[]): Promise<ExecuteResult[]>;

  /** Get the current schema version from the migration history table */
  getCurrentVersion(): Promise<string | null>;

  /** Create the migration history table if it doesn't exist */
  ensureHistoryTable(): Promise<void>;

  /** Record a migration execution in the history table */
  recordExecution(record: HistoryRecord): Promise<void>;

  /** Check if a table exists */
  tableExists(schema: string, table: string): Promise<boolean>;

  /** Get advisory lock to prevent concurrent migrations */
  acquireLock(lockId: string, timeoutMs: number): Promise<boolean>;
  releaseLock(lockId: string): Promise<void>;
}

export interface ExecuteResult {
  rowCount: number;
  executionTimeMs: number;
}

export interface HistoryRecord {
  version: string;
  description: string;
  checksum: string;
  executionTimeMs: number;
  success: boolean;
  executedBy: string;
}

export function createAdapter(engine: DatabaseEngine): EngineAdapter {
  switch (engine) {
    case 'postgresql': return new PostgresAdapter();
    case 'mysql': return new MysqlAdapter();
    case 'sqlite': return new SqliteAdapter();
  }
}
```

**Testing**:
- **test-adapter-factory**: Call `createAdapter('postgresql')`, verify it returns a `PostgresAdapter` instance with `engine === 'postgresql'`.
- **test-adapter-factory-invalid**: Call `createAdapter('oracle' as any)`, verify it throws a descriptive error.

---

#### 2.2 — PostgreSQL Adapter with Schema Introspection

**What**: Implement the PostgreSQL adapter using `pg` driver, querying `information_schema` and `pg_catalog` for full schema introspection.

**Design**:

```typescript
// packages/core/src/engines/postgresql.ts

import pg from 'pg';
import type { EngineAdapter, ConnectionConfig, SchemaData, TableDefinition, ColumnDefinition } from '../types';

export class PostgresAdapter implements EngineAdapter {
  readonly engine = 'postgresql' as const;
  private pool: pg.Pool | null = null;

  async connect(config: ConnectionConfig): Promise<void> {
    this.pool = new pg.Pool({
      host: config.host,
      port: config.port ?? 5432,
      database: config.database,
      // credentials resolved from secretRef at connection time
    });
  }

  async introspect(): Promise<SchemaData> {
    const schema = this.getSchema();
    const tables = await this.introspectTables(schema);
    const enums = await this.introspectEnums(schema);
    const extensions = await this.introspectExtensions();
    const functions = await this.introspectFunctions(schema);
    return { tables, enums, extensions, functions };
  }

  private async introspectTables(schema: string): Promise<Record<string, TableDefinition>> {
    // Query information_schema.columns for all columns
    const columnsQuery = `
      SELECT
        c.table_name,
        c.column_name,
        c.data_type,
        c.udt_name,
        c.character_maximum_length,
        c.is_nullable,
        c.column_default,
        c.numeric_precision,
        c.numeric_scale
      FROM information_schema.columns c
      WHERE c.table_schema = $1
        AND c.table_name NOT IN ('_schemashift_history', '_schemashift_lock')
      ORDER BY c.table_name, c.ordinal_position
    `;

    // Query pg_indexes for index definitions
    const indexesQuery = `
      SELECT
        schemaname, tablename, indexname, indexdef
      FROM pg_indexes
      WHERE schemaname = $1
    `;

    // Query information_schema.table_constraints + key_column_usage for constraints
    const constraintsQuery = `
      SELECT
        tc.table_name,
        tc.constraint_name,
        tc.constraint_type,
        kcu.column_name,
        ccu.table_name AS referenced_table,
        ccu.column_name AS referenced_column,
        rc.delete_rule,
        rc.update_rule
      FROM information_schema.table_constraints tc
      JOIN information_schema.key_column_usage kcu
        ON tc.constraint_name = kcu.constraint_name
        AND tc.table_schema = kcu.table_schema
      LEFT JOIN information_schema.constraint_column_usage ccu
        ON tc.constraint_name = ccu.constraint_name
        AND tc.table_schema = ccu.table_schema
      LEFT JOIN information_schema.referential_constraints rc
        ON tc.constraint_name = rc.constraint_name
      WHERE tc.table_schema = $1
      ORDER BY tc.table_name, tc.constraint_name
    `;

    // Execute all three queries and assemble into Record<string, TableDefinition>
    // ... (implementation assembles columns, indexes, constraints per table)
  }

  async ensureHistoryTable(): Promise<void> {
    await this.execute(`
      CREATE TABLE IF NOT EXISTS _schemashift_history (
        installed_rank  SERIAL PRIMARY KEY,
        version         VARCHAR(50) NOT NULL,
        description     VARCHAR(500) NOT NULL,
        checksum        VARCHAR(64) NOT NULL,
        executed_by     VARCHAR(255) NOT NULL,
        execution_time_ms INTEGER NOT NULL,
        success         BOOLEAN NOT NULL,
        installed_on    TIMESTAMPTZ NOT NULL DEFAULT now()
      )
    `);
  }

  async acquireLock(lockId: string, timeoutMs: number): Promise<boolean> {
    // Use pg_advisory_lock with a hash of the lockId
    const hash = this.hashLockId(lockId);
    const result = await this.pool!.query(
      `SELECT pg_try_advisory_lock($1) AS acquired`,
      [hash]
    );
    return result.rows[0].acquired;
  }

  async releaseLock(lockId: string): Promise<void> {
    const hash = this.hashLockId(lockId);
    await this.pool!.query(`SELECT pg_advisory_unlock($1)`, [hash]);
  }
}
```

**Testing**:
- **test-pg-introspect-empty**: Connect to empty PostgreSQL container. Introspect. Expected: `{ tables: {}, enums: {}, extensions: [], functions: {} }`.
- **test-pg-introspect-table**: Create table `CREATE TABLE users (id UUID PRIMARY KEY, email VARCHAR(320) NOT NULL UNIQUE, created_at TIMESTAMPTZ DEFAULT now())`. Introspect. Expected: `tables.users.columns.id.type === 'uuid'`, `tables.users.columns.email.nullable === false`, `tables.users.columns.email.unique === true`.
- **test-pg-introspect-fk**: Create `users` and `orders` tables with FK. Introspect. Expected: `tables.orders.columns.user_id.references.table === 'users'`.
- **test-pg-introspect-enum**: Create `CREATE TYPE status AS ENUM ('active', 'inactive')`. Introspect. Expected: `enums.status.values` equals `['active', 'inactive']`.
- **test-pg-introspect-index**: Create table with composite index. Introspect. Expected: index definition includes both columns and correct type.
- **test-pg-history-table**: Call `ensureHistoryTable()` twice. Expected: no error on second call (idempotent).
- **test-pg-advisory-lock**: Acquire lock, attempt second acquire from same connection. Expected: first returns `true`, second returns `false`. Release, re-acquire. Expected: `true`.

---

#### 2.3 — MySQL Adapter with Schema Introspection

**What**: Implement the MySQL adapter using `mysql2` driver.

**Design**:

```typescript
// packages/core/src/engines/mysql.ts

import mysql from 'mysql2/promise';
import type { EngineAdapter, ConnectionConfig, SchemaData } from '../types';

export class MysqlAdapter implements EngineAdapter {
  readonly engine = 'mysql' as const;
  private pool: mysql.Pool | null = null;

  async introspect(): Promise<SchemaData> {
    const database = this.config.database;
    // MySQL uses information_schema with database instead of schema
    const columnsQuery = `
      SELECT
        TABLE_NAME, COLUMN_NAME, DATA_TYPE, COLUMN_TYPE,
        CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE, COLUMN_DEFAULT,
        COLUMN_KEY, EXTRA
      FROM information_schema.COLUMNS
      WHERE TABLE_SCHEMA = ?
      ORDER BY TABLE_NAME, ORDINAL_POSITION
    `;

    const indexesQuery = `
      SELECT
        TABLE_NAME, INDEX_NAME, COLUMN_NAME, NON_UNIQUE, INDEX_TYPE, SEQ_IN_INDEX
      FROM information_schema.STATISTICS
      WHERE TABLE_SCHEMA = ?
      ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX
    `;

    // Assemble into SchemaData (same normalized structure as PostgreSQL)
  }

  async acquireLock(lockId: string, timeoutMs: number): Promise<boolean> {
    // MySQL uses GET_LOCK()
    const [rows] = await this.pool!.query(
      `SELECT GET_LOCK(?, ?) AS acquired`,
      [lockId, Math.floor(timeoutMs / 1000)]
    );
    return (rows as any)[0].acquired === 1;
  }

  async releaseLock(lockId: string): Promise<void> {
    await this.pool!.query(`SELECT RELEASE_LOCK(?)`, [lockId]);
  }
}
```

**Testing**:
- **test-mysql-introspect-table**: Create table in MySQL container. Introspect. Expected: same normalized `SchemaData` structure as PostgreSQL adapter output.
- **test-mysql-introspect-auto-increment**: Create table with `AUTO_INCREMENT`. Expected: `columns.id.default` includes auto-increment indicator.
- **test-mysql-lock**: Acquire and release lock via `GET_LOCK()`. Expected: same behavior as PostgreSQL advisory locks.

---

#### 2.4 — SQLite Adapter with Schema Introspection

**What**: Implement the SQLite adapter using `better-sqlite3`.

**Design**:

```typescript
// packages/core/src/engines/sqlite.ts

import Database from 'better-sqlite3';
import type { EngineAdapter, ConnectionConfig, SchemaData } from '../types';

export class SqliteAdapter implements EngineAdapter {
  readonly engine = 'sqlite' as const;
  private db: Database.Database | null = null;

  async introspect(): Promise<SchemaData> {
    // SQLite uses PRAGMA table_info() and sqlite_master
    const tables = this.db!.prepare(
      `SELECT name FROM sqlite_master WHERE type = 'table' AND name NOT LIKE 'sqlite_%' AND name != '_schemashift_history'`
    ).all();

    for (const table of tables) {
      const columns = this.db!.prepare(`PRAGMA table_info('${table.name}')`).all();
      const indexes = this.db!.prepare(`PRAGMA index_list('${table.name}')`).all();
      const fks = this.db!.prepare(`PRAGMA foreign_key_list('${table.name}')`).all();
      // Assemble into SchemaData
    }
  }

  async acquireLock(_lockId: string, _timeoutMs: number): Promise<boolean> {
    // SQLite is single-writer; use file-based lock
    this.db!.exec('BEGIN EXCLUSIVE');
    return true;
  }
}
```

**Testing**:
- **test-sqlite-introspect-table**: Create table with columns, PK, and FK. Introspect. Expected: normalized `SchemaData` matches expected structure.
- **test-sqlite-introspect-without-rowid**: Create `WITHOUT ROWID` table. Introspect. Expected: table is correctly represented.

---

## Phase 3: Schema Diffing Engine and Migration Generation

### Purpose

Build the schema diff engine that compares a declared (desired) schema against an actual (introspected) schema and generates the SQL migration plan. This is the core value proposition of the declarative mode — "describe what you want, the tool computes the migration."

### Tasks

#### 3.1 — SQL Parser Integration

**What**: Parse SQL schema files into the `SchemaData` AST representation using `pg-query-emscripten` and `sql-parser-cst`.

**Design**:

```typescript
// packages/core/src/parser/sql-parser.ts

import { parse as pgParse } from 'pg-query-emscripten';
import { parse as cstParse } from 'sql-parser-cst';
import type { SchemaData, DatabaseEngine } from '../types';

export function parseSchemaFile(sql: string, engine: DatabaseEngine): SchemaData {
  switch (engine) {
    case 'postgresql':
      return parsePostgresSchema(sql);
    case 'mysql':
      return parseMysqlSchema(sql);
    case 'sqlite':
      return parseSqliteSchema(sql);
  }
}

function parsePostgresSchema(sql: string): SchemaData {
  const ast = pgParse(sql);
  const schemaData: SchemaData = { tables: {}, enums: {}, extensions: [], functions: {} };

  for (const stmt of ast.stmts) {
    const node = stmt.stmt;
    if (node.CreateStmt) {
      // Extract table name, columns, constraints from CreateStmt AST
      const tableName = node.CreateStmt.relation.relname;
      schemaData.tables[tableName] = extractTableDefinition(node.CreateStmt);
    } else if (node.CreateEnumStmt) {
      const enumName = node.CreateEnumStmt.typeName.join('.');
      schemaData.enums[enumName] = { values: node.CreateEnumStmt.vals.map(v => v.String.sval) };
    }
    // ... handle other DDL types
  }
  return schemaData;
}
```

**Testing**:
- **test-parse-create-table**: Parse `CREATE TABLE users (id UUID PRIMARY KEY, email VARCHAR(320) NOT NULL)`. Expected: `tables.users.columns` contains `id` and `email` with correct types.
- **test-parse-fk**: Parse `CREATE TABLE orders (..., FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE)`. Expected: `columns.user_id.references` is populated.
- **test-parse-enum**: Parse `CREATE TYPE status AS ENUM ('a', 'b')`. Expected: `enums.status.values` is `['a', 'b']`.
- **test-parse-composite-index**: Parse `CREATE INDEX idx ON users (last_name, first_name)`. Expected: index definition has both columns.
- **test-parse-multistatement**: Parse a file with 10 CREATE TABLE statements. Expected: all 10 tables are in `tables`.

---

#### 3.2 — Schema Diff Algorithm

**What**: Compute the structural difference between two `SchemaData` objects and produce an ordered list of `DiffOperation` items.

**Design**:

```typescript
// packages/core/src/differ/schema-differ.ts

import type { SchemaData, TableDefinition, ColumnDefinition } from '../types';

export interface DiffOperation {
  type: DiffOperationType;
  objectType: 'table' | 'column' | 'index' | 'constraint' | 'enum' | 'function';
  schema: string;
  objectName: string;
  parentName?: string;   // table name for column operations
  before: unknown | null;
  after: unknown | null;
  isDestructive: boolean;
  riskLevel: 'low' | 'medium' | 'high' | 'critical';
}

export type DiffOperationType =
  | 'create_table' | 'drop_table' | 'rename_table'
  | 'add_column' | 'drop_column' | 'alter_column' | 'rename_column'
  | 'add_index' | 'drop_index'
  | 'add_constraint' | 'drop_constraint'
  | 'create_enum' | 'alter_enum' | 'drop_enum'
  | 'create_function' | 'alter_function' | 'drop_function';

export function diffSchemas(declared: SchemaData, actual: SchemaData): DiffOperation[] {
  const operations: DiffOperation[] = [];

  // 1. Tables present in declared but not in actual → create
  for (const [tableName, tableDef] of Object.entries(declared.tables)) {
    if (!(tableName in actual.tables)) {
      operations.push({
        type: 'create_table', objectType: 'table', schema: 'public',
        objectName: tableName, before: null, after: tableDef,
        isDestructive: false, riskLevel: 'low',
      });
      continue;
    }
    // 2. Table exists in both → diff columns, indexes, constraints
    const actualTable = actual.tables[tableName];
    operations.push(...diffColumns(tableName, tableDef, actualTable));
    operations.push(...diffIndexes(tableName, tableDef, actualTable));
    operations.push(...diffConstraints(tableName, tableDef, actualTable));
  }

  // 3. Tables present in actual but not in declared → drop (destructive)
  for (const tableName of Object.keys(actual.tables)) {
    if (!(tableName in declared.tables)) {
      operations.push({
        type: 'drop_table', objectType: 'table', schema: 'public',
        objectName: tableName, before: actual.tables[tableName], after: null,
        isDestructive: true, riskLevel: 'critical',
      });
    }
  }

  // 4. Diff enums
  operations.push(...diffEnums(declared.enums, actual.enums));

  // 5. Order operations: creates before alters, alters before drops
  //    FK-referenced tables before referencing tables
  return topologicalSort(operations);
}

function diffColumns(
  tableName: string,
  declared: TableDefinition,
  actual: TableDefinition
): DiffOperation[] {
  const ops: DiffOperation[] = [];

  for (const [colName, colDef] of Object.entries(declared.columns)) {
    if (!(colName in actual.columns)) {
      ops.push({
        type: 'add_column', objectType: 'column', schema: 'public',
        objectName: colName, parentName: tableName,
        before: null, after: colDef,
        isDestructive: false, riskLevel: colDef.nullable ? 'low' : 'medium',
      });
    } else {
      const actualCol = actual.columns[colName];
      if (!columnsEqual(colDef, actualCol)) {
        ops.push({
          type: 'alter_column', objectType: 'column', schema: 'public',
          objectName: colName, parentName: tableName,
          before: actualCol, after: colDef,
          isDestructive: isColumnAlterDestructive(actualCol, colDef),
          riskLevel: computeColumnAlterRisk(actualCol, colDef),
        });
      }
    }
  }

  for (const colName of Object.keys(actual.columns)) {
    if (!(colName in declared.columns)) {
      ops.push({
        type: 'drop_column', objectType: 'column', schema: 'public',
        objectName: colName, parentName: tableName,
        before: actual.columns[colName], after: null,
        isDestructive: true, riskLevel: 'high',
      });
    }
  }

  return ops;
}
```

**Testing**:
- **test-diff-empty-to-table**: Diff `declared: { tables: { users: {...} } }` vs `actual: { tables: {} }`. Expected: one `create_table` operation.
- **test-diff-add-column**: Diff with `declared` having one extra column. Expected: one `add_column` operation.
- **test-diff-drop-column**: Diff with `actual` having a column not in `declared`. Expected: one `drop_column` operation, `isDestructive: true`.
- **test-diff-alter-column-type**: Column type changes from `varchar(255)` to `varchar(320)`. Expected: one `alter_column` with correct before/after.
- **test-diff-no-changes**: Identical schemas. Expected: empty array.
- **test-diff-topological-order**: Table B references table A. Create both. Expected: `create_table A` comes before `create_table B` in the output.
- **test-diff-drop-table-critical**: Dropping a table is `riskLevel: 'critical'`.

---

#### 3.3 — SQL Generation from Diff Operations

**What**: Convert `DiffOperation[]` into engine-specific SQL strings.

**Design**:

```typescript
// packages/core/src/generator/sql-generator.ts

import type { DiffOperation, DatabaseEngine } from '../types';

export interface GeneratedMigration {
  sqlUp: string;
  sqlDown: string | null;   // null if any operation is irreversible
  operations: DiffOperation[];
  isDestructive: boolean;
  riskLevel: 'low' | 'medium' | 'high' | 'critical';
}

export function generateMigration(
  operations: DiffOperation[],
  engine: DatabaseEngine
): GeneratedMigration {
  const generator = getGenerator(engine);
  const upStatements: string[] = [];
  const downStatements: string[] = [];
  let anyIrreversible = false;

  for (const op of operations) {
    upStatements.push(generator.generateUp(op));
    const down = generator.generateDown(op);
    if (down === null) {
      anyIrreversible = true;
    } else {
      downStatements.unshift(down); // reverse order for rollback
    }
  }

  return {
    sqlUp: upStatements.join('\n\n'),
    sqlDown: anyIrreversible ? null : downStatements.join('\n\n'),
    operations,
    isDestructive: operations.some(op => op.isDestructive),
    riskLevel: maxRiskLevel(operations),
  };
}

// PostgreSQL-specific SQL generation
class PostgresGenerator {
  generateUp(op: DiffOperation): string {
    switch (op.type) {
      case 'create_table':
        return this.generateCreateTable(op);
      case 'add_column':
        return `ALTER TABLE "${op.parentName}" ADD COLUMN "${op.objectName}" ${this.columnDef(op.after)};`;
      case 'drop_column':
        return `ALTER TABLE "${op.parentName}" DROP COLUMN "${op.objectName}";`;
      case 'alter_column':
        return this.generateAlterColumn(op);
      case 'add_index':
        return this.generateCreateIndex(op);
      // ... other operations
    }
  }

  generateDown(op: DiffOperation): string | null {
    switch (op.type) {
      case 'create_table':
        return `DROP TABLE IF EXISTS "${op.objectName}";`;
      case 'add_column':
        return `ALTER TABLE "${op.parentName}" DROP COLUMN "${op.objectName}";`;
      case 'drop_column':
        // Irreversible — data is lost
        return null;
      case 'drop_table':
        // Could regenerate CREATE TABLE, but data is lost
        return null;
      case 'alter_column':
        // Reverse the type change
        return `ALTER TABLE "${op.parentName}" ALTER COLUMN "${op.objectName}" TYPE ${(op.before as any).type};`;
      // ...
    }
  }
}
```

**Testing**:
- **test-gen-create-table-pg**: Generate SQL for `create_table` operation. Expected: valid `CREATE TABLE` statement with correct column types, constraints.
- **test-gen-add-column-pg**: Generate SQL for `add_column`. Expected: `ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(20);`.
- **test-gen-drop-column-irreversible**: Generate migration for `drop_column`. Expected: `sqlDown` is `null`.
- **test-gen-roundtrip**: Parse a schema, diff against empty, generate SQL, execute against PostgreSQL container, introspect, compare to original. Expected: introspected schema matches declared schema.
- **test-gen-mysql-syntax**: Generate `add_column` for MySQL. Expected: uses backticks instead of double quotes.

---

#### 3.4 — Checksum Computation

**What**: Implement SHA-256 checksum computation for migration SQL and schema declarations.

**Design**:

```typescript
// packages/core/src/checksum/checksum.ts

import { createHash } from 'node:crypto';

export function computeChecksum(content: string): string {
  return createHash('sha256')
    .update(normalizeWhitespace(content))
    .digest('hex');
}

function normalizeWhitespace(sql: string): string {
  return sql
    .replace(/--.*$/gm, '')        // strip line comments
    .replace(/\/\*[\s\S]*?\*\//g, '') // strip block comments
    .replace(/\s+/g, ' ')          // collapse whitespace
    .trim();
}
```

**Testing**:
- **test-checksum-deterministic**: Same SQL produces same checksum across invocations.
- **test-checksum-whitespace-invariant**: `CREATE TABLE x (id INT);` and `CREATE  TABLE  x  ( id  INT );` produce the same checksum.
- **test-checksum-comment-invariant**: SQL with and without comments produces the same checksum.
- **test-checksum-different-sql**: Different SQL produces different checksums.

---

## Phase 4: CLI Foundation — Init, Plan, Migrate, Status

### Purpose

Build the CLI commands that provide the core user workflow: initialize a project, preview a migration plan, apply migrations, and check status. After this phase, a developer can use the tool end-to-end via the command line.

### Tasks

#### 4.1 — Configuration File Loading

**What**: Load and validate `schemashift.toml` configuration files.

**Design**:

```toml
# schemashift.toml (example)

[project]
name = "my-app"
migrations_dir = "db/migrations"
schema_file = "db/schema.sql"
naming_convention = "timestamp"

[targets.development]
engine = "postgresql"
tier = "development"

[targets.development.connection]
host = "localhost"
port = 5432
database = "myapp_dev"
schema = "public"

[targets.production]
engine = "postgresql"
tier = "production"

[targets.production.connection]
host = "db.example.com"
port = 5432
database = "myapp_prod"
secret_ref = "vault://secret/db/prod"

[targets.production.policies]
requires_approval = true
min_approvals = 2
required_roles = ["dba"]
auto_approve_risk = ["low"]
expand_contract_enabled = true
```

```typescript
// packages/cli/src/config/loader.ts

import { parse as parseTOML } from '@iarna/toml';
import { z } from 'zod';
import type { SchemashiftConfig } from '@schemashift/core';

const configSchema = z.object({
  project: z.object({
    name: z.string(),
    migrations_dir: z.string().default('migrations/'),
    schema_file: z.string().default('schema.sql'),
    naming_convention: z.enum(['timestamp', 'sequential']).default('timestamp'),
  }),
  targets: z.record(z.object({
    engine: z.enum(['postgresql', 'mysql', 'sqlite']),
    tier: z.enum(['development', 'staging', 'production']).default('development'),
    connection: z.object({
      host: z.string().optional(),
      port: z.number().optional(),
      database: z.string(),
      schema: z.string().optional(),
      secret_ref: z.string().optional(),
    }),
    policies: z.object({
      requires_approval: z.boolean().default(false),
      min_approvals: z.number().default(1),
      required_roles: z.array(z.string()).default([]),
      auto_approve_risk: z.array(z.string()).default(['low']),
      expand_contract_enabled: z.boolean().default(false),
    }).optional(),
  })),
});

export async function loadConfig(dir: string): Promise<SchemashiftConfig> {
  const configPath = path.join(dir, 'schemashift.toml');
  const raw = await fs.readFile(configPath, 'utf-8');
  const parsed = parseTOML(raw);
  return configSchema.parse(parsed);
}
```

**Testing**:
- **test-config-load-valid**: Load the example TOML. Expected: parsed config matches expected structure.
- **test-config-load-defaults**: Load minimal TOML (just project name and one target). Expected: defaults are applied.
- **test-config-load-missing**: Load from directory without `schemashift.toml`. Expected: descriptive error message.
- **test-config-load-invalid-engine**: Engine is `'oracle'`. Expected: Zod validation error listing valid engines.

---

#### 4.2 — `schemashift init` Command

**What**: Initialize a new project directory with a `schemashift.toml`, `migrations/` directory, and optional `schema.sql`.

**Design**:

```typescript
// packages/cli/src/commands/init.ts

import { Command } from 'commander';

export const initCommand = new Command('init')
  .description('Initialize a new SchemaShift project')
  .option('--engine <engine>', 'Default database engine', 'postgresql')
  .option('--dir <dir>', 'Project directory', '.')
  .action(async (options) => {
    const dir = path.resolve(options.dir);
    // 1. Create schemashift.toml with engine-appropriate defaults
    // 2. Create migrations/ directory
    // 3. Create schema.sql with header comment
    // 4. Print success message with next-steps instructions
  });
```

**Testing**:
- **test-init-creates-files**: Run `init` in a temp directory. Expected: `schemashift.toml`, `migrations/`, and `schema.sql` all exist.
- **test-init-no-overwrite**: Run `init` in a directory that already has `schemashift.toml`. Expected: error with "already initialized" message.
- **test-init-custom-engine**: Run `init --engine mysql`. Expected: TOML file contains `engine = "mysql"`.

---

#### 4.3 — `schemashift plan` Command

**What**: Compute and display the migration plan (diff between declared schema and live database) without applying it.

**Design**:

```typescript
// packages/cli/src/commands/plan.ts

export const planCommand = new Command('plan')
  .description('Preview the migration plan for a target')
  .argument('[target]', 'Target name', 'development')
  .option('--format <format>', 'Output format', 'table')  // table | json | sql
  .action(async (target, options) => {
    const config = await loadConfig('.');
    const targetConfig = config.targets[target];
    const adapter = createAdapter(targetConfig.engine);
    await adapter.connect(targetConfig.connection);

    // 1. Parse the declared schema from schema.sql
    const declaredSql = await fs.readFile(config.project.schema_file, 'utf-8');
    const declared = parseSchemaFile(declaredSql, targetConfig.engine);

    // 2. Introspect the live database
    const actual = await adapter.introspect();

    // 3. Compute the diff
    const operations = diffSchemas(declared, actual);

    // 4. Generate the migration SQL
    const migration = generateMigration(operations, targetConfig.engine);

    // 5. Display the plan
    if (options.format === 'json') {
      console.log(JSON.stringify(migration, null, 2));
    } else if (options.format === 'sql') {
      console.log(migration.sqlUp);
    } else {
      renderPlanTable(migration);  // ink-based formatted table
    }

    await adapter.disconnect();
  });
```

**Testing**:
- **test-plan-no-changes**: Schema matches database. Expected: "No changes detected" message.
- **test-plan-add-table**: Schema has a table not in database. Expected: plan shows `CREATE TABLE` operation.
- **test-plan-json-output**: Run with `--format json`. Expected: valid JSON with `sqlUp`, `sqlDown`, `operations` fields.
- **test-plan-destructive-warning**: Plan includes a `DROP COLUMN`. Expected: output includes risk warning.

---

#### 4.4 — `schemashift migrate` Command

**What**: Apply pending migrations to a target database, with locking, checksum validation, and history recording.

**Design**:

```typescript
// packages/cli/src/commands/migrate.ts

export const migrateCommand = new Command('migrate')
  .description('Apply pending migrations to a target')
  .argument('[target]', 'Target name', 'development')
  .option('--dry-run', 'Show what would be applied without executing')
  .option('--version <version>', 'Apply up to a specific version')
  .action(async (target, options) => {
    const config = await loadConfig('.');
    const targetConfig = config.targets[target];
    const adapter = createAdapter(targetConfig.engine);
    await adapter.connect(targetConfig.connection);

    // 1. Acquire advisory lock
    const lockAcquired = await adapter.acquireLock(`schemashift:${target}`, 30_000);
    if (!lockAcquired) {
      console.error('Another migration is in progress. Aborting.');
      process.exit(1);
    }

    try {
      // 2. Ensure history table exists
      await adapter.ensureHistoryTable();

      // 3. Read migration files from migrations/ directory
      const migrationFiles = await readMigrationFiles(config.project.migrations_dir);

      // 4. Get current version
      const currentVersion = await adapter.getCurrentVersion();

      // 5. Filter to pending migrations
      const pending = filterPending(migrationFiles, currentVersion, options.version);

      // 6. Validate checksums against history
      validateChecksums(migrationFiles, await adapter.getHistory());

      // 7. Apply each migration in order
      for (const mig of pending) {
        if (options.dryRun) {
          console.log(`[DRY RUN] Would apply: ${mig.version} — ${mig.description}`);
          continue;
        }
        const startTime = Date.now();
        try {
          await adapter.executeInTransaction([mig.sqlUp]);
          const elapsed = Date.now() - startTime;
          await adapter.recordExecution({
            version: mig.version,
            description: mig.description,
            checksum: mig.checksum,
            executionTimeMs: elapsed,
            success: true,
            executedBy: os.userInfo().username,
          });
          console.log(`✓ Applied ${mig.version} (${elapsed}ms)`);
        } catch (err) {
          await adapter.recordExecution({
            version: mig.version, description: mig.description,
            checksum: mig.checksum, executionTimeMs: Date.now() - startTime,
            success: false, executedBy: os.userInfo().username,
          });
          console.error(`✗ Failed ${mig.version}: ${err.message}`);
          process.exit(1);
        }
      }
    } finally {
      await adapter.releaseLock(`schemashift:${target}`);
      await adapter.disconnect();
    }
  });
```

**Testing**:
- **test-migrate-apply-single**: Create one migration file. Run `migrate`. Expected: migration applied, history table has one record with `success: true`.
- **test-migrate-apply-multiple**: Three migration files. Run `migrate`. Expected: all three applied in version order.
- **test-migrate-idempotent**: Run `migrate` twice. Expected: second run reports "no pending migrations."
- **test-migrate-checksum-mismatch**: Modify a migration file after applying it. Run `migrate`. Expected: error about checksum mismatch.
- **test-migrate-dry-run**: Run with `--dry-run`. Expected: no SQL executed, no history records created.
- **test-migrate-failure-stops**: Three migrations; second one has invalid SQL. Expected: first applied, second fails, third not attempted.
- **test-migrate-lock**: Start two concurrent `migrate` processes. Expected: one succeeds, other reports lock error.

---

#### 4.5 — `schemashift status` Command

**What**: Display the current migration status: which migrations have been applied, which are pending, and whether drift is detected.

**Design**:

```typescript
// packages/cli/src/commands/status.ts

export const statusCommand = new Command('status')
  .description('Show migration status for a target')
  .argument('[target]', 'Target name', 'development')
  .option('--format <format>', 'Output format', 'table')
  .action(async (target, options) => {
    // 1. Load config and connect
    // 2. Read migration files from disk
    // 3. Read history from _schemashift_history table
    // 4. Cross-reference: which are applied, pending, failed
    // 5. Display as table or JSON
    // Output columns: Version | Description | Status | Applied At | Time
  });
```

**Testing**:
- **test-status-no-history**: Fresh database with no history table. Expected: all migrations shown as "pending."
- **test-status-partial**: Two of three migrations applied. Expected: two "applied", one "pending."
- **test-status-json**: Run with `--format json`. Expected: valid JSON array of migration statuses.

---

## Phase 5: Declarative Mode and Drift Detection

### Purpose

Implement the "desired state" workflow (declare target schema, tool computes the migration) and drift detection (compare live database against declared state). After this phase, the tool supports both imperative (migration files) and declarative (schema.sql) workflows.

### Tasks

#### 5.1 — `schemashift apply` Command (Declarative Mode)

**What**: Read the declared schema file, diff against live database, generate migration plan, and apply it.

**Design**:

```typescript
// packages/cli/src/commands/apply.ts

export const applyCommand = new Command('apply')
  .description('Apply the declared schema to a target (declarative mode)')
  .argument('[target]', 'Target name', 'development')
  .option('--auto-approve', 'Skip confirmation prompt')
  .option('--save-migration', 'Save the generated migration to the migrations/ directory')
  .action(async (target, options) => {
    const config = await loadConfig('.');
    const targetConfig = config.targets[target];
    const adapter = createAdapter(targetConfig.engine);
    await adapter.connect(targetConfig.connection);

    // 1. Parse declared schema
    const declared = parseSchemaFile(
      await fs.readFile(config.project.schema_file, 'utf-8'),
      targetConfig.engine
    );

    // 2. Introspect live database
    const actual = await adapter.introspect();

    // 3. Diff
    const operations = diffSchemas(declared, actual);
    if (operations.length === 0) {
      console.log('Schema is up to date. No changes needed.');
      return;
    }

    // 4. Generate migration
    const migration = generateMigration(operations, targetConfig.engine);

    // 5. Display plan and confirm
    renderPlanTable(migration);
    if (!options.autoApprove) {
      const confirmed = await promptConfirm('Apply these changes?');
      if (!confirmed) return;
    }

    // 6. Optionally save migration file
    if (options.saveMigration) {
      const version = generateVersion(config.project.naming_convention);
      await saveMigrationFile(config.project.migrations_dir, version, migration);
    }

    // 7. Apply with locking
    await applyWithLock(adapter, target, migration);
  });
```

**Testing**:
- **test-apply-create-from-scratch**: Empty database, schema.sql defines two tables. Expected: both tables created.
- **test-apply-add-column**: Database has `users` table. Schema adds `phone` column. Expected: column added.
- **test-apply-save-migration**: Run with `--save-migration`. Expected: migration file written to `migrations/` with timestamp version.
- **test-apply-no-changes**: Schema matches database. Expected: "No changes needed" message.
- **test-apply-destructive-confirm**: Plan includes DROP. Expected: confirmation prompt appears (test with `--auto-approve` to verify the apply path).

---

#### 5.2 — Drift Detection Engine

**What**: Compare the declared schema against the live database and report divergences as `DriftFinding[]`.

**Design**:

```typescript
// packages/core/src/differ/drift-detector.ts

import type { SchemaData, DriftReport, DriftFinding, DriftType } from '../types';

export function detectDrift(
  declared: SchemaData,
  actual: SchemaData
): DriftFinding[] {
  const findings: DriftFinding[] = [];

  // Objects in declared but missing from actual database
  for (const [tableName, tableDef] of Object.entries(declared.tables)) {
    if (!(tableName in actual.tables)) {
      findings.push({
        objectType: 'table', schema: 'public', table: tableName,
        driftType: 'missing_in_db',
        expected: JSON.stringify(tableDef), actual: null,
        severity: 'error', isResolved: false,
      });
      continue;
    }
    // Column-level drift
    for (const [colName, colDef] of Object.entries(tableDef.columns)) {
      const actualCol = actual.tables[tableName]?.columns?.[colName];
      if (!actualCol) {
        findings.push({
          objectType: 'column', schema: 'public', table: tableName, column: colName,
          driftType: 'missing_in_db', expected: colDef.type, actual: null,
          severity: 'error', isResolved: false,
        });
      } else if (colDef.type !== actualCol.type) {
        findings.push({
          objectType: 'column', schema: 'public', table: tableName, column: colName,
          driftType: 'column_type_mismatch', expected: colDef.type, actual: actualCol.type,
          severity: 'warning', isResolved: false,
        });
      }
    }
    // Objects in actual but not in declared (unexpected objects in DB)
    for (const colName of Object.keys(actual.tables[tableName].columns)) {
      if (!(colName in tableDef.columns)) {
        findings.push({
          objectType: 'column', schema: 'public', table: tableName, column: colName,
          driftType: 'missing_in_declaration', expected: null,
          actual: actual.tables[tableName].columns[colName].type,
          severity: 'warning', isResolved: false,
        });
      }
    }
  }

  // Tables in actual but not in declared
  for (const tableName of Object.keys(actual.tables)) {
    if (!(tableName in declared.tables)) {
      findings.push({
        objectType: 'table', schema: 'public', table: tableName,
        driftType: 'missing_in_declaration', expected: null,
        actual: JSON.stringify(actual.tables[tableName]),
        severity: 'warning', isResolved: false,
      });
    }
  }

  return findings;
}
```

**Testing**:
- **test-drift-clean**: Identical schemas. Expected: empty findings array.
- **test-drift-missing-table**: Declared has table `orders` not in database. Expected: finding with `driftType: 'missing_in_db'`, `severity: 'error'`.
- **test-drift-extra-column**: Database has column `legacy_id` not in declaration. Expected: `driftType: 'missing_in_declaration'`.
- **test-drift-type-mismatch**: Declared column is `varchar(255)`, actual is `text`. Expected: `driftType: 'column_type_mismatch'`.
- **test-drift-multiple**: Multiple drift issues. Expected: all findings returned.

---

#### 5.3 — `schemashift drift` Command

**What**: CLI command to run drift detection and display results.

**Design**:

```typescript
// packages/cli/src/commands/drift.ts

export const driftCommand = new Command('drift')
  .description('Detect schema drift between declared state and live database')
  .argument('[target]', 'Target name', 'development')
  .option('--format <format>', 'Output format', 'table')
  .option('--fail-on-drift', 'Exit with code 1 if drift is detected (for CI)')
  .action(async (target, options) => {
    // 1. Load config, connect, parse declared, introspect actual
    // 2. Run detectDrift()
    // 3. Display findings
    // 4. If --fail-on-drift and findings.length > 0, process.exit(1)
  });
```

**Testing**:
- **test-drift-cli-clean**: No drift. Expected: "No drift detected" message, exit code 0.
- **test-drift-cli-found**: Drift exists. Expected: findings displayed, exit code 0.
- **test-drift-cli-fail-flag**: Drift exists with `--fail-on-drift`. Expected: exit code 1.
- **test-drift-cli-json**: Run with `--format json`. Expected: valid JSON array of findings.

---

## Phase 6: Rollback and Compensating Migrations

### Purpose

Implement rollback execution for reversible migrations and compensating migration generation for irreversible DDL changes. This phase addresses the "Intelligent Rollback Planning" differentiator from the research.

### Tasks

#### 6.1 — `schemashift rollback` Command

**What**: Roll back the most recently applied migration or a specific version.

**Design**:

```typescript
// packages/cli/src/commands/rollback.ts

export const rollbackCommand = new Command('rollback')
  .description('Roll back the last applied migration')
  .argument('[target]', 'Target name', 'development')
  .option('--to <version>', 'Roll back to a specific version (exclusive)')
  .option('--dry-run', 'Show rollback SQL without executing')
  .action(async (target, options) => {
    // 1. Load history; find the last applied migration(s)
    // 2. Read the migration file(s); check if sql_down exists
    // 3. If sql_down is null, report "irreversible" and suggest compensating migration
    // 4. Otherwise, apply sql_down in reverse order
    // 5. Update history table (mark as rolled_back)
  });
```

**Testing**:
- **test-rollback-single**: Apply a migration with `sql_down`, roll back. Expected: database returns to pre-migration state. History shows `rolled_back`.
- **test-rollback-to-version**: Apply 3 migrations. Rollback `--to V001`. Expected: V003 and V002 rolled back, V001 remains.
- **test-rollback-irreversible**: Migration has no `sql_down`. Expected: error message suggesting `schemashift compensate`.
- **test-rollback-dry-run**: Run with `--dry-run`. Expected: shows SQL but does not execute.

---

#### 6.2 — Compensating Migration Generator

**What**: For irreversible DDL (DROP COLUMN, ALTER TYPE with data loss), generate a compensating migration that restores the schema to the pre-change state without restoring data.

**Design**:

```typescript
// packages/core/src/generator/compensating.ts

import type { DiffOperation, DatabaseEngine } from '../types';

export interface CompensatingMigration {
  sql: string;
  dataLossWarnings: string[];
  feasibility: 'safe' | 'data_loss' | 'impossible';
  explanation: string;
}

export function generateCompensatingMigration(
  operations: DiffOperation[],
  engine: DatabaseEngine
): CompensatingMigration {
  const statements: string[] = [];
  const warnings: string[] = [];
  let feasibility: 'safe' | 'data_loss' | 'impossible' = 'safe';

  for (const op of operations.reverse()) {
    switch (op.type) {
      case 'drop_column': {
        // Can recreate the column but data is lost
        const col = op.before as ColumnDefinition;
        statements.push(
          `ALTER TABLE "${op.parentName}" ADD COLUMN "${op.objectName}" ${col.type}${col.nullable ? '' : ' NOT NULL DEFAULT <FILL_IN>'};`
        );
        warnings.push(`Data for ${op.parentName}.${op.objectName} cannot be restored`);
        feasibility = 'data_loss';
        break;
      }
      case 'drop_table': {
        // Can recreate structure but data is lost
        statements.push(`-- Recreate table ${op.objectName} — DATA IS LOST`);
        statements.push(generateCreateTableFromDef(op.before, engine));
        warnings.push(`All data in table ${op.objectName} is permanently lost`);
        feasibility = 'data_loss';
        break;
      }
      case 'alter_column': {
        // Type change: reverse may truncate data
        const before = op.before as ColumnDefinition;
        statements.push(
          `ALTER TABLE "${op.parentName}" ALTER COLUMN "${op.objectName}" TYPE ${before.type};`
        );
        if (isNarrowingTypeChange(op.after as ColumnDefinition, before)) {
          warnings.push(`Reverting type of ${op.parentName}.${op.objectName} may truncate data`);
          feasibility = 'data_loss';
        }
        break;
      }
    }
  }

  return {
    sql: statements.join('\n'),
    dataLossWarnings: warnings,
    feasibility,
    explanation: feasibility === 'safe'
      ? 'This compensating migration can be safely applied.'
      : `WARNING: This compensating migration involves data loss:\n${warnings.map(w => `  - ${w}`).join('\n')}`,
  };
}
```

**Testing**:
- **test-compensate-drop-column**: Generate for a `drop_column`. Expected: `ADD COLUMN` with original type, `feasibility: 'data_loss'`, warning about data loss.
- **test-compensate-drop-table**: Generate for a `drop_table`. Expected: `CREATE TABLE` reconstructed, `feasibility: 'data_loss'`.
- **test-compensate-alter-column**: Generate for narrowing `varchar(320)` to `varchar(100)`. Expected: `ALTER COLUMN TYPE varchar(320)`, warning about truncation.
- **test-compensate-safe**: Generate for `add_column` reversal (which is just `DROP COLUMN`). Expected: `feasibility: 'safe'`.

---

## Phase 7: Platform Database and Server API

### Purpose

Set up the platform's own database (using the Hybrid JSONB data model from Suggestion 3), build the Hono API server, and implement the core CRUD endpoints. This transforms the CLI-only tool into a platform with state persistence, multi-user support, and API access.

### Tasks

#### 7.1 — Platform Database Schema (Drizzle)

**What**: Define the platform's own database schema using Drizzle ORM, matching Data Model Suggestion 3.

**Design**:

```typescript
// packages/server/src/db/schema.ts

import { pgTable, uuid, varchar, text, boolean, integer, timestamp, jsonb, uniqueIndex, index } from 'drizzle-orm/pg-core';

export const workspaces = pgTable('workspaces', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  plan: varchar('plan', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const members = pgTable('members', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id, { onDelete: 'cascade' }),
  email: varchar('email', { length: 320 }).notNull(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  role: varchar('role', { length: 50 }).notNull().default('developer'),
  profile: jsonb('profile').notNull().default({}),
  isActive: boolean('is_active').notNull().default(true),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceEmail: uniqueIndex('idx_members_workspace_email').on(table.workspaceId, table.email),
  workspaceIdx: index('idx_members_workspace').on(table.workspaceId),
}));

export const projects = pgTable('projects', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  description: text('description'),
  config: jsonb('config').notNull().default({}),
  isArchived: boolean('is_archived').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceSlug: uniqueIndex('idx_projects_workspace_slug').on(table.workspaceId, table.slug),
}));

export const targets = pgTable('targets', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 100 }).notNull(),
  tier: varchar('tier', { length: 50 }).notNull().default('development'),
  engine: varchar('engine', { length: 50 }).notNull(),
  connection: jsonb('connection').notNull().default({}),
  policies: jsonb('policies').notNull().default({}),
  currentVersion: varchar('current_version', { length: 50 }),
  lastConnectedAt: timestamp('last_connected_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  projectName: uniqueIndex('idx_targets_project_name').on(table.projectId, table.name),
}));

export const schemaSnapshots = pgTable('schema_snapshots', {
  id: uuid('id').primaryKey().defaultRandom(),
  targetId: uuid('target_id').notNull().references(() => targets.id, { onDelete: 'cascade' }),
  snapshotType: varchar('snapshot_type', { length: 50 }).notNull(),
  schemaData: jsonb('schema_data').notNull(),
  contentHash: varchar('content_hash', { length: 64 }).notNull(),
  capturedAt: timestamp('captured_at', { withTimezone: true }).notNull().defaultNow(),
  capturedBy: uuid('captured_by').references(() => members.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const migrations = pgTable('migrations', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  version: varchar('version', { length: 50 }).notNull(),
  description: varchar('description', { length: 500 }).notNull(),
  migrationType: varchar('migration_type', { length: 50 }).notNull().default('versioned'),
  source: varchar('source', { length: 50 }).notNull().default('manual'),
  sqlUp: text('sql_up').notNull(),
  sqlDown: text('sql_down'),
  checksum: varchar('checksum', { length: 64 }).notNull(),
  analysis: jsonb('analysis').notNull().default({}),
  engineOptions: jsonb('engine_options').notNull().default({}),
  authoredBy: uuid('authored_by').notNull().references(() => members.id),
  commitSha: varchar('commit_sha', { length: 40 }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  projectVersion: uniqueIndex('idx_migrations_project_version').on(table.projectId, table.version),
}));

export const migrationRuns = pgTable('migration_runs', {
  id: uuid('id').primaryKey().defaultRandom(),
  migrationId: uuid('migration_id').notNull().references(() => migrations.id),
  targetId: uuid('target_id').notNull().references(() => targets.id),
  status: varchar('status', { length: 50 }).notNull().default('pending'),
  executedSql: text('executed_sql'),
  executionTimeMs: integer('execution_time_ms'),
  executedBy: uuid('executed_by').references(() => members.id),
  executedAt: timestamp('executed_at', { withTimezone: true }),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  details: jsonb('details').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const driftReports = pgTable('drift_reports', {
  id: uuid('id').primaryKey().defaultRandom(),
  targetId: uuid('target_id').notNull().references(() => targets.id),
  scanType: varchar('scan_type', { length: 50 }).notNull().default('scheduled'),
  status: varchar('status', { length: 50 }).notNull().default('clean'),
  findings: jsonb('findings').notNull().default([]),
  driftCount: integer('drift_count').notNull().default(0),
  scannedAt: timestamp('scanned_at', { withTimezone: true }).notNull().defaultNow(),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const auditEvents = pgTable('audit_events', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id),
  actorId: uuid('actor_id').references(() => members.id),
  action: varchar('action', { length: 100 }).notNull(),
  resourceType: varchar('resource_type', { length: 100 }).notNull(),
  resourceId: uuid('resource_id').notNull(),
  context: jsonb('context').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const notificationRules = pgTable('notification_rules', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  config: jsonb('config').notNull(),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- **test-schema-migration**: Run Drizzle migration against PostgreSQL container. Expected: all 11 tables created.
- **test-schema-fk-integrity**: Insert a `member` with invalid `workspace_id`. Expected: FK constraint violation.
- **test-schema-unique-constraints**: Insert duplicate workspace slug. Expected: unique constraint violation.

---

#### 7.2 — Hono API Server with Core CRUD Endpoints

**What**: Build the REST API using Hono with OpenAPI 3.1 schema generation.

**Design**:

```typescript
// packages/server/src/api/routes/projects.ts

import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';

const createProjectSchema = z.object({
  name: z.string().min(1).max(255),
  slug: z.string().min(1).max(100).regex(/^[a-z0-9-]+$/),
  description: z.string().optional(),
  config: z.object({
    vcs: z.object({
      provider: z.enum(['github', 'gitlab', 'bitbucket']).optional(),
      repoUrl: z.string().url().optional(),
      branch: z.string().default('main'),
      migrationPath: z.string().default('db/migrations/'),
    }).optional(),
  }).optional(),
});

export const projectRoutes = new Hono()
  .post('/', zValidator('json', createProjectSchema), async (c) => {
    // Create project in database
    // Return 201 with created project
  })
  .get('/', async (c) => {
    // List projects for workspace
    // Return 200 with paginated list
  })
  .get('/:projectId', async (c) => {
    // Get project by ID
    // Return 200 or 404
  })
  .get('/:projectId/migrations', async (c) => {
    // List migrations for project
  })
  .post('/:projectId/migrations', async (c) => {
    // Create a migration (upload SQL)
  })
  .post('/:projectId/targets/:targetId/migrate', async (c) => {
    // Trigger migration execution for a target
  })
  .post('/:projectId/targets/:targetId/drift', async (c) => {
    // Trigger drift scan for a target
  });
```

API routes summary:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces` | Create workspace |
| GET | `/api/workspaces/:id` | Get workspace |
| POST | `/api/workspaces/:id/members` | Add member |
| POST | `/api/projects` | Create project |
| GET | `/api/projects` | List projects |
| GET | `/api/projects/:id` | Get project |
| POST | `/api/projects/:id/targets` | Create target |
| GET | `/api/projects/:id/targets` | List targets |
| POST | `/api/projects/:id/migrations` | Upload migration |
| GET | `/api/projects/:id/migrations` | List migrations |
| POST | `/api/projects/:id/targets/:tid/migrate` | Execute migration |
| POST | `/api/projects/:id/targets/:tid/drift` | Run drift scan |
| GET | `/api/projects/:id/targets/:tid/drift` | Get drift reports |
| GET | `/api/projects/:id/targets/:tid/status` | Get target status |

**Testing**:
- **test-api-create-project**: POST valid project. Expected: 201 with project ID.
- **test-api-create-project-invalid**: POST with missing name. Expected: 400 with validation errors.
- **test-api-list-projects**: Create 3 projects. GET list. Expected: 200 with 3 items.
- **test-api-get-project-404**: GET non-existent project. Expected: 404.
- **test-api-create-migration**: POST migration SQL. Expected: 201 with checksum computed.
- **test-api-trigger-drift**: POST drift scan. Expected: 202 with scan ID.

---

#### 7.3 — Authentication with OAuth 2.0

**What**: Implement OAuth 2.0 / OIDC authentication using the `arctic` library for GitHub, Google, and generic OIDC providers.

**Design**:

```typescript
// packages/server/src/auth/oauth.ts

import { GitHub, Google, generateState, generateCodeVerifier } from 'arctic';

export function createGitHubProvider(clientId: string, clientSecret: string): GitHub {
  return new GitHub(clientId, clientSecret, null);
}

// packages/server/src/auth/middleware.ts

import { createMiddleware } from 'hono/factory';

export const authMiddleware = createMiddleware(async (c, next) => {
  const token = c.req.header('Authorization')?.replace('Bearer ', '');
  if (!token) {
    return c.json({ error: 'Unauthorized' }, 401);
  }

  // Validate JWT or lookup API token hash
  const member = await validateToken(token);
  if (!member) {
    return c.json({ error: 'Invalid token' }, 401);
  }

  c.set('member', member);
  c.set('workspaceId', member.workspaceId);
  await next();
});
```

**Testing**:
- **test-auth-missing-token**: Request without Authorization header. Expected: 401.
- **test-auth-invalid-token**: Request with invalid token. Expected: 401.
- **test-auth-valid-token**: Request with valid API token. Expected: 200, `c.get('member')` is populated.
- **test-auth-api-token-scopes**: Token with `migrations:read` scope tries to POST migration. Expected: 403.

---

## Phase 8: CI/CD Integration and GitHub Action

### Purpose

Build the CI/CD integration layer: machine-readable CLI output, exit codes for pipeline gates, and a reusable GitHub Action. After this phase, teams can add migration safety checks to their CI pipelines.

### Tasks

#### 8.1 — CI Output Mode for CLI

**What**: Add `--ci` flag to all CLI commands that produces machine-readable JSON output and appropriate exit codes.

**Design**:

```typescript
// packages/cli/src/output/ci-formatter.ts

export interface CIOutput {
  status: 'pass' | 'fail' | 'warn';
  summary: string;
  checks: CICheck[];
  exitCode: number;
}

export interface CICheck {
  name: string;
  status: 'pass' | 'fail' | 'warn' | 'skip';
  message: string;
  details?: Record<string, unknown>;
}

// Exit codes:
// 0 = success (no issues)
// 1 = failure (migration would fail, checksum mismatch, etc.)
// 2 = warning (drift detected, destructive changes, high risk)

export function formatCI(checks: CICheck[]): CIOutput {
  const hasFail = checks.some(c => c.status === 'fail');
  const hasWarn = checks.some(c => c.status === 'warn');
  return {
    status: hasFail ? 'fail' : hasWarn ? 'warn' : 'pass',
    summary: hasFail ? 'Migration checks failed' : hasWarn ? 'Migration checks passed with warnings' : 'All checks passed',
    checks,
    exitCode: hasFail ? 1 : hasWarn ? 2 : 0,
  };
}
```

CI checks performed by `schemashift check --ci`:
1. **checksum_validation**: All applied migrations have matching checksums.
2. **no_destructive_without_approval**: Destructive operations are flagged.
3. **drift_detection**: No drift between declared schema and live database.
4. **migration_order**: No gaps or out-of-order migrations.
5. **rollback_available**: All pending migrations have rollback SQL (or compensating migration is feasible).

**Testing**:
- **test-ci-all-pass**: Clean state. Expected: `{ status: 'pass', exitCode: 0 }`.
- **test-ci-checksum-fail**: Tampered migration. Expected: `{ status: 'fail', exitCode: 1 }`, check `checksum_validation` is `fail`.
- **test-ci-drift-warn**: Drift detected. Expected: `{ status: 'warn', exitCode: 2 }`.
- **test-ci-json-output**: Output is valid JSON parseable by `JSON.parse()`.

---

#### 8.2 — GitHub Action

**What**: Create a reusable GitHub Action that runs `schemashift check` in PR pipelines.

**Design**:

```yaml
# .github/actions/schemashift-check/action.yml

name: 'SchemaShift Migration Check'
description: 'Validate database migrations in CI'
inputs:
  target:
    description: 'Target environment to check against'
    required: false
    default: 'development'
  database-url:
    description: 'Database connection URL (overrides schemashift.toml)'
    required: false
  fail-on-drift:
    description: 'Fail if drift is detected'
    required: false
    default: 'false'
  fail-on-destructive:
    description: 'Fail if destructive migrations are pending'
    required: false
    default: 'true'
runs:
  using: 'composite'
  steps:
    - run: npm install -g schemashift
      shell: bash
    - run: |
        schemashift check \
          --target ${{ inputs.target }} \
          --ci \
          ${{ inputs.fail-on-drift == 'true' && '--fail-on-drift' || '' }} \
          ${{ inputs.fail-on-destructive == 'true' && '--fail-on-destructive' || '' }} \
          > $GITHUB_STEP_SUMMARY
      shell: bash
      env:
        DATABASE_URL: ${{ inputs.database-url }}
```

**Testing**:
- **test-action-yaml-valid**: Parse the action.yml. Expected: valid YAML with required fields.
- **test-action-integration**: Run the composite action steps locally using `act`. Expected: `schemashift check` executes and produces output.

---

## Phase 9: Web UI — Approval Workflows and Migration Dashboard

### Purpose

Build the web-based approval workflow and migration dashboard using React. This enables DBA review before production migrations, migration history visualization, and drift monitoring — the governance features that differentiate SchemaShift from CLI-only tools like Flyway.

### Tasks

#### 9.1 — Web UI Scaffold and Layout

**What**: Set up the React application with routing, authentication, and the shell layout.

**Design**:

```typescript
// packages/web/src/main.tsx

import { createRoot } from 'react-dom/client';
import { BrowserRouter, Routes, Route } from 'react-router';
import { AuthProvider } from './hooks/useAuth';
import { Layout } from './components/Layout';
import { DashboardPage } from './pages/Dashboard';
import { MigrationsPage } from './pages/Migrations';
import { DriftPage } from './pages/Drift';
import { SettingsPage } from './pages/Settings';

createRoot(document.getElementById('root')!).render(
  <AuthProvider>
    <BrowserRouter>
      <Routes>
        <Route element={<Layout />}>
          <Route path="/" element={<DashboardPage />} />
          <Route path="/projects/:projectId/migrations" element={<MigrationsPage />} />
          <Route path="/projects/:projectId/drift" element={<DriftPage />} />
          <Route path="/projects/:projectId/settings" element={<SettingsPage />} />
        </Route>
      </Routes>
    </BrowserRouter>
  </AuthProvider>
);
```

Pages:
- **Dashboard**: Overview of all projects with migration counts, drift status, pending approvals.
- **Migrations**: List of migrations for a project with status per environment (dev/staging/prod columns). Filter by status, risk level.
- **Migration Detail**: SQL diff view, risk analysis, approval controls, execution history.
- **Drift**: Drift findings per environment. Side-by-side comparison of declared vs. actual schema.
- **Settings**: Project configuration, target management, notification rules, approval policies.

**Testing**:
- **test-web-routing**: Navigate to `/projects/123/migrations`. Expected: MigrationsPage renders.
- **test-web-auth-redirect**: Unauthenticated user navigates to `/`. Expected: redirect to login page.

---

#### 9.2 — Approval Workflow UI

**What**: Build the migration approval interface: review migration SQL, approve/reject, track approval status.

**Design**:

```typescript
// packages/web/src/components/ApprovalPanel.tsx

interface ApprovalPanelProps {
  migrationRun: MigrationRun;
  currentMember: Member;
  onApprove: (comment: string) => Promise<void>;
  onReject: (comment: string) => Promise<void>;
}

export function ApprovalPanel({ migrationRun, currentMember, onApprove, onReject }: ApprovalPanelProps) {
  // Display:
  // 1. Migration SQL (syntax-highlighted with diff view)
  // 2. Risk analysis badge (low/medium/high/critical)
  // 3. Affected objects list
  // 4. Approval status: "1 of 2 approvals received"
  // 5. Previous approval decisions with reviewer names and comments
  // 6. Approve / Reject buttons (disabled if current user already reviewed)
  // 7. Comment text area
}
```

API integration:
- `POST /api/projects/:id/targets/:tid/runs/:runId/approve` — approve with comment
- `POST /api/projects/:id/targets/:tid/runs/:runId/reject` — reject with comment

Server-side approval logic:
```typescript
// packages/server/src/services/approval.ts

export async function submitApproval(
  runId: string,
  reviewerId: string,
  decision: 'approve' | 'reject',
  comment: string
): Promise<MigrationRun> {
  const run = await db.query.migrationRuns.findFirst({ where: eq(migrationRuns.id, runId) });
  if (run.status !== 'awaiting_approval') throw new Error('Not awaiting approval');

  const details = run.details as MigrationRunDetails;
  details.approvals.push({
    reviewerId, reviewerName: reviewer.displayName,
    decision, comment, decidedAt: new Date(),
  });

  if (decision === 'reject') {
    await db.update(migrationRuns).set({ status: 'rejected', details }).where(eq(migrationRuns.id, runId));
  } else if (details.approvals.filter(a => a.decision === 'approve').length >= details.requiredApprovals) {
    await db.update(migrationRuns).set({ status: 'approved', details }).where(eq(migrationRuns.id, runId));
  } else {
    await db.update(migrationRuns).set({ details }).where(eq(migrationRuns.id, runId));
  }

  // Record audit event
  await recordAuditEvent('approval.submitted', 'migration_run', runId, { decision, comment });

  return updatedRun;
}
```

**Testing**:
- **test-approval-approve**: Submit approval. Expected: run status changes to `approved` when threshold met.
- **test-approval-reject**: Submit rejection. Expected: run status changes to `rejected`.
- **test-approval-insufficient**: One approval when two required. Expected: status stays `awaiting_approval`.
- **test-approval-duplicate**: Same reviewer approves twice. Expected: error "already reviewed."
- **test-approval-wrong-role**: Developer tries to approve when only DBAs can. Expected: 403.

---

## Phase 10: Multi-Environment Promotion and Audit Trail

### Purpose

Implement the environment promotion workflow (dev -> staging -> production) with promotion gates, and the ISO 27001 / SOC 2 audit trail. After this phase, the platform tracks every state change and supports regulated-industry compliance requirements.

### Tasks

#### 10.1 — Environment Promotion Engine

**What**: Build the promotion workflow that tracks migration state across environments and enforces promotion order.

**Design**:

```typescript
// packages/server/src/services/promotion.ts

export interface PromotionCheck {
  canPromote: boolean;
  blockers: string[];
  warnings: string[];
}

export async function checkPromotion(
  projectId: string,
  migrationId: string,
  targetEnvironmentId: string
): Promise<PromotionCheck> {
  const target = await db.query.targets.findFirst({ where: eq(targets.id, targetEnvironmentId) });
  const migration = await db.query.migrations.findFirst({ where: eq(migrations.id, migrationId) });

  const blockers: string[] = [];
  const warnings: string[] = [];

  // 1. Check promotion order: must be applied in all lower-tier environments first
  const lowerTierTargets = await db.query.targets.findMany({
    where: and(
      eq(targets.projectId, projectId),
      lt(targets.promotionOrder, target.promotionOrder) // using raw SQL or drizzle operator
    ),
  });

  for (const lowerTarget of lowerTierTargets) {
    const run = await findSuccessfulRun(migrationId, lowerTarget.id);
    if (!run) {
      blockers.push(`Migration must be applied to ${lowerTarget.name} before ${target.name}`);
    }
  }

  // 2. Check approval policy for target environment
  const policies = target.policies as TargetPolicies;
  if (policies.requiresApproval) {
    // Approval will be checked during execution
  }

  // 3. Check maintenance window for production
  if (target.tier === 'production' && policies.maintenanceWindow) {
    if (!isWithinMaintenanceWindow(policies.maintenanceWindow)) {
      warnings.push(`Outside maintenance window (${policies.maintenanceWindow.day} ${policies.maintenanceWindow.start}-${policies.maintenanceWindow.end} ${policies.maintenanceWindow.timezone})`);
    }
  }

  return { canPromote: blockers.length === 0, blockers, warnings };
}
```

**Testing**:
- **test-promotion-order**: Try to promote to production without applying to staging first. Expected: `canPromote: false`, blocker message.
- **test-promotion-valid**: Applied in dev and staging. Expected: `canPromote: true`.
- **test-promotion-maintenance-window**: Outside maintenance window. Expected: warning but `canPromote: true`.

---

#### 10.2 — Audit Trail Service

**What**: Implement the audit event recording service that captures every state change for ISO 27001 / SOC 2 compliance.

**Design**:

```typescript
// packages/server/src/services/audit.ts

export type AuditAction =
  | 'workspace.created' | 'workspace.updated'
  | 'member.invited' | 'member.role_changed' | 'member.removed'
  | 'project.created' | 'project.archived'
  | 'target.created' | 'target.connection_updated'
  | 'migration.created' | 'migration.updated'
  | 'migration.execution_started' | 'migration.execution_succeeded'
  | 'migration.execution_failed' | 'migration.rolled_back'
  | 'approval.requested' | 'approval.approved' | 'approval.rejected'
  | 'drift.scan_started' | 'drift.detected' | 'drift.resolved'
  | 'token.created' | 'token.revoked';

export async function recordAuditEvent(
  action: AuditAction,
  resourceType: string,
  resourceId: string,
  context: Record<string, unknown>,
  actor?: { memberId: string; ipAddress?: string; userAgent?: string }
): Promise<void> {
  await db.insert(auditEvents).values({
    workspaceId: context.workspaceId as string,
    actorId: actor?.memberId ?? null,
    action,
    resourceType,
    resourceId,
    context: {
      ...context,
      ip_address: actor?.ipAddress,
      user_agent: actor?.userAgent,
      timestamp: new Date().toISOString(),
    },
  });
}

// Query: get audit trail for a resource
export async function getAuditTrail(
  resourceType: string,
  resourceId: string,
  options: { limit?: number; offset?: number; since?: Date } = {}
): Promise<AuditEvent[]> {
  return db.query.auditEvents.findMany({
    where: and(
      eq(auditEvents.resourceType, resourceType),
      eq(auditEvents.resourceId, resourceId),
      options.since ? gte(auditEvents.createdAt, options.since) : undefined,
    ),
    orderBy: desc(auditEvents.createdAt),
    limit: options.limit ?? 50,
    offset: options.offset ?? 0,
  });
}
```

**Testing**:
- **test-audit-record**: Record an audit event. Expected: event persisted with all fields.
- **test-audit-trail-query**: Record 5 events for a resource. Query trail. Expected: 5 events returned in reverse chronological order.
- **test-audit-trail-since**: Query with `since` filter. Expected: only events after the date.
- **test-audit-immutability**: Attempt to UPDATE an audit event row. Expected: application-layer guard prevents it (or database trigger rejects it).

---

## Phase 11: AI-Powered Features

### Purpose

Integrate the AI layer for the platform's key differentiators: impact analysis, natural-language migration generation, and drift root-cause analysis. These features use the Claude API with prompt caching for cost efficiency.

### Tasks

#### 11.1 — AI Impact Analyzer

**What**: Analyze migration SQL to identify all ORM models, queries, and application code that reference affected schema objects.

**Design**:

```typescript
// packages/ai/src/impact-analyzer.ts

import Anthropic from '@anthropic-ai/sdk';
import type { Migration, MigrationAnalysis, AffectedObject, CodeReference } from '@schemashift/core';

const client = new Anthropic();

export interface ImpactAnalysisInput {
  migration: Migration;
  schemaSnapshot: SchemaData;
  codebaseFiles: CodeFile[];   // relevant source files from VCS
}

export interface CodeFile {
  path: string;
  content: string;
  language: string;
}

export async function analyzeImpact(input: ImpactAnalysisInput): Promise<MigrationAnalysis> {
  const affectedObjects = input.migration.analysis.affectedObjects;
  const affectedNames = affectedObjects.map(o => `${o.schema}.${o.name}`).join(', ');

  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'text',
            text: `You are a database migration safety analyzer. Analyze the following migration and codebase to identify all code that references the affected schema objects.

MIGRATION SQL:
${input.migration.sqlUp}

AFFECTED OBJECTS: ${affectedNames}

CURRENT SCHEMA:
${JSON.stringify(input.schemaSnapshot, null, 2)}

CODEBASE FILES:
${input.codebaseFiles.map(f => `--- ${f.path} (${f.language}) ---\n${f.content}`).join('\n\n')}

Respond with JSON:
{
  "risk_score": <0.0 to 1.0>,
  "orm_references": [{"file": "...", "line": <n>, "type": "orm_field|raw_query|api_endpoint"}],
  "downstream_conflicts": ["description..."],
  "recommendation": "plain language recommendation",
  "risk_factors": [{"type": "...", "object": "...", "severity": "low|medium|high|critical", "description": "..."}]
}`,
            cache_control: { type: 'ephemeral' },
          },
        ],
      },
    ],
  });

  const result = JSON.parse(response.content[0].text);
  return {
    ...input.migration.analysis,
    impact: {
      riskScore: result.risk_score,
      ormReferences: result.orm_references,
      downstreamConflicts: result.downstream_conflicts,
      recommendation: result.recommendation,
    },
    riskLevel: scoreToRiskLevel(result.risk_score),
    aiConfidence: result.risk_score,
    analyzedAt: new Date(),
  };
}

function scoreToRiskLevel(score: number): RiskLevel {
  if (score >= 0.8) return 'critical';
  if (score >= 0.5) return 'high';
  if (score >= 0.2) return 'medium';
  return 'low';
}
```

**Testing**:
- **test-impact-orm-reference**: Migration drops column `users.email`. Codebase has Prisma model with `email` field. Expected: `orm_references` includes the file and line.
- **test-impact-raw-query**: Codebase has `SELECT email FROM users`. Expected: `orm_references` includes with type `raw_query`.
- **test-impact-no-references**: Migration adds a new table with no existing code references. Expected: `risk_score` near 0, empty `orm_references`.
- **test-impact-json-validity**: Response from AI is valid JSON matching the expected schema.

---

#### 11.2 — Natural-Language Migration Generator

**What**: Generate migration SQL from a plain-language description of the desired change.

**Design**:

```typescript
// packages/ai/src/migration-generator.ts

export interface GenerateFromNLInput {
  description: string;         // "Add a phone number field to users, optional, max 20 chars"
  currentSchema: SchemaData;
  engine: DatabaseEngine;
  conventions: {
    namingConvention: 'snake_case' | 'camelCase';
    indexNaming: string;       // e.g., 'idx_{table}_{columns}'
  };
}

export interface GeneratedMigrationFromNL {
  sqlUp: string;
  sqlDown: string | null;
  explanation: string;         // human-readable explanation of what was generated
  confidence: number;          // 0.0 to 1.0
  warnings: string[];
}

export async function generateFromNaturalLanguage(
  input: GenerateFromNLInput
): Promise<GeneratedMigrationFromNL> {
  const response = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'text',
            text: `You are a database migration SQL generator. Given the current schema and a natural-language description, generate the migration SQL.

DATABASE ENGINE: ${input.engine}
NAMING CONVENTION: ${input.conventions.namingConvention}
INDEX NAMING: ${input.conventions.indexNaming}

CURRENT SCHEMA:
${JSON.stringify(input.currentSchema, null, 2)}

REQUESTED CHANGE:
${input.description}

Generate the safest possible migration. If the change could cause data loss or downtime, use expand-contract pattern. Include both UP and DOWN SQL.

Respond with JSON:
{
  "sql_up": "...",
  "sql_down": "..." or null,
  "explanation": "plain language explanation",
  "confidence": <0.0 to 1.0>,
  "warnings": ["..."]
}`,
            cache_control: { type: 'ephemeral' },
          },
        ],
      },
    ],
  });

  return JSON.parse(response.content[0].text);
}
```

**Testing**:
- **test-nlgen-add-column**: Description: "Add a phone number to users, optional". Expected: `ALTER TABLE users ADD COLUMN phone VARCHAR(20)` or similar.
- **test-nlgen-rename-column**: Description: "Rename email to email_address in users". Expected: safe expand-contract SQL (add new column, copy data, drop old) or `ALTER TABLE ... RENAME COLUMN`.
- **test-nlgen-with-rollback**: Generated migration includes non-null `sql_down`.
- **test-nlgen-confidence**: Confidence is between 0.0 and 1.0.

---

#### 11.3 — Drift Root-Cause Analyzer

**What**: When drift is detected, correlate drift findings with deployment history, migration executions, and audit events to identify the root cause.

**Design**:

```typescript
// packages/ai/src/drift-analyzer.ts

export interface DriftRootCauseInput {
  findings: DriftFinding[];
  recentMigrationRuns: MigrationRun[];
  recentAuditEvents: AuditEvent[];
  recentDeployments?: DeploymentRecord[];
}

export interface DriftRootCause {
  finding: DriftFinding;
  probableCause: string;
  confidence: number;
  correlatedEvents: string[];  // descriptions of correlated events
  remediation: string;
}

export async function analyzeDriftRootCause(
  input: DriftRootCauseInput
): Promise<DriftRootCause[]> {
  // Uses Claude to correlate drift findings with recent events
  // Returns probable cause for each finding
}
```

**Testing**:
- **test-drift-root-cause-manual-ddl**: Drift finding for an extra index. Audit log shows direct SQL execution. Expected: `probableCause` identifies the manual DDL.
- **test-drift-root-cause-failed-migration**: Column type mismatch. Migration run shows failure. Expected: `probableCause` identifies the failed migration.

---

## Phase 12: Zero-Downtime Migrations and Advanced Features

### Purpose

Implement expand-contract migration pattern for zero-downtime schema changes on large tables, notification system, and Flyway/Liquibase migration import. These are the polishing features that make the platform production-ready for enterprise use.

### Tasks

#### 12.1 — Expand-Contract Migration Pattern

**What**: Implement the zero-downtime migration pattern where schema changes are split into expand (add new) and contract (remove old) phases with a data migration step in between.

**Design**:

```typescript
// packages/core/src/generator/expand-contract.ts

export interface ExpandContractPlan {
  expandMigration: GeneratedMigration;    // Phase 1: add new column/table
  dataMigrationSql: string;               // Phase 2: backfill data
  contractMigration: GeneratedMigration;  // Phase 3: drop old column/table
  viewMigration?: string;                 // Optional: create view for backward compat
}

export function planExpandContract(
  operation: DiffOperation,
  engine: DatabaseEngine
): ExpandContractPlan {
  switch (operation.type) {
    case 'alter_column': {
      // Example: rename column users.email → users.email_address
      const table = operation.parentName!;
      const oldCol = operation.objectName;
      const newDef = operation.after as ColumnDefinition;

      return {
        expandMigration: {
          sqlUp: `ALTER TABLE "${table}" ADD COLUMN "${oldCol}_new" ${newDef.type};`,
          sqlDown: `ALTER TABLE "${table}" DROP COLUMN "${oldCol}_new";`,
          operations: [operation],
          isDestructive: false,
          riskLevel: 'low',
        },
        dataMigrationSql: `UPDATE "${table}" SET "${oldCol}_new" = "${oldCol}";`,
        contractMigration: {
          sqlUp: [
            `ALTER TABLE "${table}" DROP COLUMN "${oldCol}";`,
            `ALTER TABLE "${table}" RENAME COLUMN "${oldCol}_new" TO "${oldCol}";`,
          ].join('\n'),
          sqlDown: null,    // irreversible
          operations: [operation],
          isDestructive: true,
          riskLevel: 'high',
        },
      };
    }
    case 'drop_column': {
      // Expand: mark column as deprecated (add comment)
      // Contract: actually drop after confirmation period
      // ...
    }
  }
}
```

**Testing**:
- **test-ec-rename-column**: Plan expand-contract for column rename. Expected: 3-phase plan with expand (add new), data migration (copy), contract (drop old, rename new).
- **test-ec-type-change**: Plan for type change `varchar(255)` to `text`. Expected: expand adds new column with `text` type, data migration copies with cast.
- **test-ec-execute-full-cycle**: Execute all three phases against PostgreSQL container. Expected: column renamed with zero data loss, application queries work during each phase.

---

#### 12.2 — Notification System

**What**: Send notifications via Slack, email, and webhooks when migration events occur.

**Design**:

```typescript
// packages/server/src/services/notifications.ts

export interface NotificationPayload {
  eventType: string;
  project: { id: string; name: string };
  target: { id: string; name: string; tier: string };
  migration?: { version: string; description: string; riskLevel: string };
  message: string;
  url: string;   // link to the web UI
}

export async function sendNotifications(
  projectId: string,
  payload: NotificationPayload
): Promise<void> {
  const rules = await db.query.notificationRules.findMany({
    where: and(
      eq(notificationRules.projectId, projectId),
      eq(notificationRules.isActive, true),
    ),
  });

  for (const rule of rules) {
    const config = rule.config as NotificationConfig;
    if (!config.events.includes(payload.eventType)) continue;

    switch (config.channel) {
      case 'slack':
        await sendSlackNotification(config.webhookUrl, payload);
        break;
      case 'webhook':
        await sendWebhookNotification(config.webhookUrl, payload);
        break;
      case 'email':
        await sendEmailNotification(config.recipients, payload);
        break;
    }
  }
}
```

**Testing**:
- **test-notify-slack**: Send Slack notification with mock webhook server. Expected: POST to webhook URL with Slack block format.
- **test-notify-filter**: Rule subscribes to `migration_failed` only. Send `migration_succeeded`. Expected: no notification sent.
- **test-notify-webhook**: Send webhook notification. Expected: POST with JSON body containing all payload fields.

---

#### 12.3 — Migration Import (Flyway and Liquibase)

**What**: Import existing migration histories from Flyway (`flyway_schema_history`) and Liquibase (`DATABASECHANGELOG`) tables.

**Design**:

```typescript
// packages/cli/src/commands/import.ts

export const importCommand = new Command('import')
  .description('Import migration history from Flyway or Liquibase')
  .argument('<source>', 'Source tool: flyway | liquibase')
  .argument('[target]', 'Target name', 'development')
  .action(async (source, target) => {
    const adapter = await connectToTarget(target);

    if (source === 'flyway') {
      // Read from flyway_schema_history table
      const history = await adapter.execute(
        `SELECT installed_rank, version, description, type, script, checksum, installed_by, installed_on, execution_time, success
         FROM flyway_schema_history ORDER BY installed_rank`
      );
      // Convert to SchemaShift migration format
      // Write migration files to migrations/ directory
      // Record in _schemashift_history
    } else if (source === 'liquibase') {
      // Read from DATABASECHANGELOG table
      const history = await adapter.execute(
        `SELECT ID, AUTHOR, FILENAME, DATEEXECUTED, ORDEREXECUTED, MD5SUM, DESCRIPTION, TAG, LIQUIBASE
         FROM DATABASECHANGELOG ORDER BY ORDEREXECUTED`
      );
      // Convert to SchemaShift format
    }
  });
```

**Testing**:
- **test-import-flyway**: Create a `flyway_schema_history` table with 3 entries. Run import. Expected: 3 migration files created, history recorded.
- **test-import-liquibase**: Create a `DATABASECHANGELOG` table with entries. Run import. Expected: migrations imported correctly.
- **test-import-idempotent**: Run import twice. Expected: second run detects already-imported migrations and skips.

---

## Phase Summary & Dependencies

```
Phase 1: Project Scaffold & Core Types
    │
    ├──► Phase 2: Engine Adapters & Schema Introspection
    │       │
    │       ├──► Phase 3: Schema Diffing & Migration Generation
    │       │       │
    │       │       ├──► Phase 4: CLI Foundation (init, plan, migrate, status)
    │       │       │       │
    │       │       │       ├──► Phase 5: Declarative Mode & Drift Detection
    │       │       │       │       │
    │       │       │       │       └──► Phase 6: Rollback & Compensating Migrations
    │       │       │       │
    │       │       │       └──► Phase 8: CI/CD Integration & GitHub Action
    │       │       │
    │       │       └──► Phase 12: Zero-Downtime (Expand-Contract) ←─── (can start after Phase 3)
    │       │
    │       └──► Phase 7: Platform Database & Server API
    │               │
    │               ├──► Phase 9: Web UI — Approvals & Dashboard
    │               │
    │               ├──► Phase 10: Multi-Environment Promotion & Audit Trail
    │               │
    │               └──► Phase 11: AI-Powered Features
    │
    Parallelism: Phases 8, 9, 10, 11 can proceed in parallel after Phase 7.
    Parallelism: Phase 12 can start as soon as Phase 3 completes.
```

---

## Definition of Done (per phase)

1. All tasks in the phase have been implemented and merged to main.
2. All specified test scenarios pass (`turbo test` exits with code 0).
3. TypeScript compilation succeeds with zero errors (`turbo typecheck`).
4. No new lint warnings introduced (`turbo lint`).
5. Integration tests with real database containers (PostgreSQL, MySQL, SQLite) pass where specified.
6. API endpoints (Phase 7+) have OpenAPI documentation generated and accessible at `/api/docs`.
7. CLI commands (Phase 4+) include `--help` text with usage examples.
8. All database state changes are captured in the audit trail (Phase 10+).
9. No secrets, credentials, or connection strings are committed to the repository.
10. Package builds produce valid npm-publishable artifacts.
