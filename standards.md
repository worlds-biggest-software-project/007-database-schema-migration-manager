# Standards & API Reference

> Project: Database Schema Migration Manager · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

#### ISO/IEC 25012 — Data Quality Model
- **Title:** Data Quality Model
- **Number:** ISO/IEC 25012
- **URL:** https://www.iso.org/standard/61355.html
- **Relevance:** Defines data quality characteristics (completeness, consistency, accuracy) that schema migrations must preserve to maintain database integrity and prevent data loss during schema transformations.

#### ISO/IEC 27001 — Information Security Management Systems
- **Title:** Information Security Management Systems
- **Number:** ISO/IEC 27001
- **URL:** https://www.iso.org/standard/54534.html
- **Relevance:** Establishes requirements for controlling who can apply schema changes and maintaining tamper-evident audit trails, critical for regulated industries (finance, healthcare) requiring compliance evidence.

#### ISO/IEC 19583-26 — Data Models for Metadata Registries
- **Title:** Data Models for Metadata Registries in XML Schema
- **Number:** ISO/IEC 19583-26:2026
- **URL:** https://www.iso.org/standard/83546.html
- **Relevance:** Specifies standardized representations of data models and metadata structures in machine-readable formats (XML Schema), enabling platform-neutral schema exchange across organizations.

#### ISO/IEC TS 23220-2 — Mobile Document Data Models
- **Title:** Data Models for Mobile Document Credentials and Identity Exchange
- **Number:** ISO/IEC TS 23220-2:2026
- **URL:** https://www.iso.org/standard/82892.html
- **Relevance:** Defines standardized data models for credential and identity data exchange, applicable to schema designs supporting identity and access management in migration pipelines.

#### ISO 8000 — Data Quality Standards
- **Title:** Data Quality for Reuse and Integration
- **Number:** ISO 8000
- **URL:** https://www.iso.org/standard/40001.html
- **Relevance:** Establishes principles for data quality in shared and integrated data environments, ensuring schema migrations maintain quality attributes across database boundaries.

---

### W3C & IETF Standards

#### RFC 7231 — HTTP/1.1 Semantics and Content
- **Title:** Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content
- **Number:** RFC 7231
- **URL:** https://datatracker.ietf.org/doc/html/rfc7231
- **Relevance:** Defines HTTP semantics essential for REST API design in database migration tools; establishes method semantics, status codes, and content negotiation for migration endpoints.

#### RFC 8288 — Web Linking
- **Title:** Web Linking
- **Number:** RFC 8288
- **URL:** https://datatracker.ietf.org/doc/html/rfc8288
- **Relevance:** Defines link relation types and serialization for RESTful APIs; applicable to discovering related migration resources and versioning links in migration workflows.

#### JSON Schema (IETF Internet-Draft)
- **Title:** JSON Schema (Internet-Draft)
- **Number:** IETF JSON Schema Charter
- **URL:** https://datatracker.ietf.org/doc/charter-ietf-jsonschema/
- **Relevance:** Machine-readable schema validation language widely adopted by OpenAPI, GraphQL, and other API specifications for defining database entity structures and migration changesets.

#### W3C JSON-LD — Linked Data for JSON
- **Title:** JSON for Linking Data (JSON-LD)
- **URL:** https://www.w3.org/ns/json-ld
- **Relevance:** Enables semantic enrichment of schema definitions and migration metadata, allowing systems to understand relationships between database entities across migrations.

---

### Data Model & API Specifications

#### OpenAPI Specification 3.1
- **Title:** OpenAPI Specification Version 3.1.0
- **Version:** 3.1.0+ (latest 3.2.0)
- **URL:** https://spec.openapis.org/oas/v3.1.0.html
- **Relevance:** Industry-standard for documenting RESTful APIs in migration tools; JSON Schema 2020-12 alignment enables precise schema definitions for database entity APIs and migration endpoint contracts.

#### GraphQL Specification
- **Title:** GraphQL Query Language Specification
- **URL:** https://spec.graphql.org/
- **Relevance:** Provides strongly-typed query language for schema introspection and mutation APIs; enables clients to discover available migrations and database structure without coupling to REST endpoints.

#### Protocol Buffers (Protobuf 3)
- **Title:** Protocol Buffers Version 3 Specification
- **URL:** https://developers.google.com/protocol-buffers
- **Relevance:** Language-neutral, platform-neutral serialization format for efficient migration data exchange; widely adopted in high-performance migration tools for gRPC APIs.

#### Apache Avro
- **Title:** Apache Avro Schema Definition Format
- **URL:** https://avro.apache.org/docs/current/spec.html
- **Relevance:** Schema-aware data serialization enabling schema evolution and versioning; useful for changesets and migration metadata transport in event-driven migration systems.

---

### Security & Authentication Standards

#### OAuth 2.0 / OpenID Connect
- **Title:** OAuth 2.0 Authorization Framework + OpenID Connect Core
- **URL:** https://oauth.net/2/ and https://openid.net/connect/
- **Relevance:** Standard authentication/authorization mechanisms for migration API access control; enables fine-grained permission scoping (e.g., "apply migrations to production") and audit-trail token binding.

#### OWASP Security Standards
- **Title:** OWASP Top 10 and Secure Coding Practices
- **URL:** https://owasp.org/www-project-top-ten/
- **Relevance:** Addresses injection attacks (SQL injection in migration scripts), insecure direct object references, and API security patterns critical for migration tooling handling DDL/DML statements.

#### NIST Cybersecurity Framework
- **Title:** NIST Cybersecurity Framework (CSF)
- **URL:** https://www.nist.gov/cyberframework
- **Relevance:** Provides risk management and governance framework for database change control; maps to "Identify," "Protect," and "Detect" functions for migration policy enforcement and incident response.

---

### API Governance & Interoperability Standards

#### OpenGitOps v1.0 — GitOps Specification
- **Title:** OpenGitOps: GitOps Principles and Glossary (v1.0)
- **URL:** https://opengitops.dev/
- **Relevance:** Defines declarative, version-controlled, automatically-reconciled infrastructure operations; directly applicable to schema-as-code and GitOps-native migration workflows (used by Atlas, SchemaHero, Bytebase).

#### CNCF Specifications & Standards
- **Title:** Cloud Native Computing Foundation (CNCF) Landscape & Standards
- **URL:** https://www.cncf.io/
- **Relevance:** Alignment with Kubernetes Operators (SchemaHero), gRPC APIs (Bytebase), and containerized deployment patterns; establishes ecosystem expectations for cloud-native migration tools.

---

### API Quality & Compliance Standards

#### Semantic Versioning 2.0.0
- **Title:** Semantic Versioning Specification (SemVer 2.0.0)
- **URL:** https://semver.org/
- **Relevance:** Establishes version numbering convention for migration script ordering and API versioning; enables safe dependency resolution and backward compatibility signaling in migration frameworks.

#### DORA (DevOps Research and Assessment) Metrics
- **Title:** DORA Metrics: Deployment Frequency, Lead Time for Changes, Change Failure Rate, Time to Restore Service
- **URL:** https://dora.dev/guides/dora-metrics/
- **Relevance:** Quantifies migration quality through change failure rate and time to restore metrics; enables data-driven assessment of migration tool effectiveness and risk management.

---

## Similar Products — Developer Documentation & APIs

### Flyway by Redgate

- **Description:** SQL-first, version-controlled migration tool using numbered migration scripts; the most widely deployed migration framework globally with 14+ years of production history.
- **API Documentation:** https://documentation.red-gate.com/flyway/ (Official) | https://flywaydb.org/documentation/usage/api (Community Docs)
- **SDKs/Libraries:**
  - Java API: https://documentation.red-gate.com/fd/api-java-277579358.html
  - Spring Boot Integration: https://docs.spring.io/spring-boot/api/rest/actuator/flyway.html
  - Maven Plugin: https://flywaydb.org/documentation/usage/maven
  - Gradle Plugin: https://flywaydb.org/documentation/usage/gradle
- **Developer Guide:** https://flywaydb.org/documentation/getstarted/
- **Standards:** REST/JSON (Spring Boot Actuator endpoint), Semantic Versioning for migration naming (V1__init.sql pattern)
- **Authentication:** No built-in auth; relies on database credentials; Enterprise tier adds role-based governance

---

### Liquibase

- **Description:** XML/YAML/JSON/SQL changelog-based migration framework with strong enterprise compliance features; rebranded commercial edition to "Liquibase Secure" in 2025.
- **API Documentation:** https://docs.liquibase.com/home.html (Official) | https://docs.liquibase.com/oss/user-guide-4-33/home.html (OSS 4.33)
- **SDKs/Libraries:**
  - Java API & Core: https://www.liquibase.org/maven
  - Maven Plugin: https://docs.liquibase.com/tools-integrations/maven/home.html
  - Gradle Plugin: https://docs.liquibase.com/tools-integrations/gradle/home.html
  - Command Line: https://docs.liquibase.com/cli/home.html
  - Liquibase Pro & Secure Tiers: Enterprise features for regulated industries
- **Developer Guide:** https://docs.liquibase.com/start-here/index.html
- **Standards:** Multi-format changelogs (XML/YAML/JSON/SQL), Semantic Versioning for changeset ordering, OpenAPI (via Liquibase Hub for API integrations)
- **Authentication:** Database authentication; Secure tier adds RBAC, OAuth 2.0 for Liquibase platform, audit logging

---

### Atlas (Ariga)

- **Description:** Declarative "schema-as-code" tool that computes migration plans from desired-state definitions; includes Drift Inspector, Atlas Copilot (AI-assisted setup), and native CI/CD integration.
- **API Documentation:** https://atlasgo.io/docs (Official) | https://atlasgo.io/cli-reference (CLI Reference)
- **SDKs/Libraries:**
  - Go SDK: https://pkg.go.dev/ariga.io/atlas (Go Packages)
  - GitHub Actions: https://github.com/ariga/atlas-action
  - Kubernetes Operator: https://github.com/ariga/atlas-operator
  - Terraform Provider: https://registry.terraform.io/providers/ariga/atlas/latest
- **Developer Guide:** https://atlasgo.io/getting-started/ and https://atlasgo.io/docs/guides/
- **Standards:** Declarative HCL/SQL (schema-as-code), REST API (Atlas Cloud), OpenAPI-compatible introspection, Terraform state integration
- **Authentication:** Atlas Cloud uses API tokens and OAuth 2.0; self-hosted uses database credentials

---

### Bytebase

- **Description:** Web-based database DevSecOps platform with approval workflows, RBAC, SOC 2 audit trails, schema version control, and drift detection across 20+ database engines.
- **API Documentation:** https://www.bytebase.com/docs/api/ (Official API Docs) | https://docs.bytebase.com/introduction/what-is-bytebase (Product Docs)
- **SDKs/Libraries:**
  - REST API: https://www.bytebase.com/docs/api/overview/
  - gRPC API: Available (documented in API overview)
  - Terraform Provider: https://registry.terraform.io/providers/bytebase/bytebase/latest
  - GitHub Actions: Bytebase GitHub Actions integration for VCS-based workflows
  - Python/JavaScript/Go SDKs: Community-maintained bindings available on GitHub
- **Developer Guide:** https://docs.bytebase.com/get-started/ and https://docs.bytebase.com/api/overview/
- **Standards:** REST/JSON API, gRPC, OpenAPI 3.1, Terraform state, GitOps integrations (GitHub, GitLab, Bitbucket)
- **Authentication:** Database credentials, API tokens, OAuth 2.0, OIDC, SAML (Enterprise)

---

### Prisma Migrate

- **Description:** ORM-integrated migration tool generating SQL from Prisma schema definitions; tightly coupled to Prisma Data Platform with excellent TypeScript/Node.js developer experience.
- **API Documentation:** https://www.prisma.io/docs/orm/prisma-migrate/ (Official) | https://www.prisma.io/docs/ (Full Docs)
- **SDKs/Libraries:**
  - Node.js/TypeScript: https://www.npmjs.com/package/@prisma/client
  - Prisma CLI: https://www.prisma.io/docs/orm/tools-and-interfaces/prisma-cli
  - Docker Image: https://hub.docker.com/r/prismarelational/prisma
  - Package Manager: npm, yarn, pnpm, bun compatible
- **Developer Guide:** https://www.prisma.io/docs/getting-started/setup-prisma/
- **Standards:** Prisma Schema Definition Language (proprietary), OpenAPI generation support, SQL standard compliance (database-agnostic layer)
- **Authentication:** Database connection strings; Prisma Platform uses API tokens and OAuth 2.0

---

### Sqitch

- **Description:** Pure-SQL, dependency-aware migration tool using change/deploy/revert/verify structure with Merkle tree integrity checking; no commercial component, no UI, Perl-based.
- **API Documentation:** https://sqitch.org/docs/ (Official) | https://metacpan.org/dist/App-Sqitch (Perl CPAN API)
- **SDKs/Libraries:**
  - Perl API: https://metacpan.org/dist/App-Sqitch/view/lib/sqitch.pod
  - CLI: Standalone binary, no external dependencies
  - Git Integration: Native Git-like command structure
  - Database Drivers: PostgreSQL, MySQL, Firebird, Exasol, Oracle, Snowflake, SQLite, Vertica, Yugabyte, CockroachDB
- **Developer Guide:** https://sqitch.org/docs/manual/sqitch-intro/ and per-database tutorials
- **Standards:** Pure SQL (no DSL), SemVer-compatible plan files, Git-inspired workflows, cryptographic checksums (Merkle tree)
- **Authentication:** Database-level authentication; no built-in API authentication (CLI-based)

---

### Harness Database DevOps

- **Description:** Enterprise platform integrating database schema changes into CI/CD pipelines with automated rollback, policy enforcement, and GitOps-native workflows; part of broader Harness platform.
- **API Documentation:** https://developer.harness.io/docs/database-devops/ (Official) | https://developer.harness.io/docs/api/ (General Harness API)
- **SDKs/Libraries:**
  - REST API: https://developer.harness.io/docs/database-devops/overview/ (Overview; full API endpoint docs in developer hub)
  - Terraform Provider: Enterprise integration available
  - GitHub Actions: CI/CD pipeline templates for DB DevOps workflows
  - CLI/UI: Web-based console + CLI tools
- **Developer Guide:** https://developer.harness.io/docs/database-devops/overview/
- **Standards:** REST API, OpenAPI (Harness API spec), Terraform state, CI/CD pipeline DSL (Harness Pipelines)
- **Authentication:** Harness API tokens, OAuth 2.0, SAML/OIDC (Enterprise)

---

### pgroll (by Xata)

- **Description:** Postgres-specific tool enabling zero-downtime multi-version migrations via expand-contract pattern; immature but production-proven ecosystem with single Go binary deployment.
- **API Documentation:** https://xataio.github.io/pgroll/ (Official) | https://github.com/xataio/pgroll (GitHub Source)
- **SDKs/Libraries:**
  - Go Binary: Single cross-platform executable at https://github.com/xataio/pgroll/releases
  - CLI: Native command-line tool (expand/contract workflow)
  - Docker: Official Docker image available
  - Node.js/Python SDKs: Community-maintained bindings
- **Developer Guide:** https://neon.com/guides/pgroll (Neon integration guide) | https://github.com/xataio/pgroll#usage
- **Standards:** PostgreSQL-native SQL, expand-contract DDL pattern, pure CLI (no REST API officially)
- **Authentication:** PostgreSQL connection credentials; no built-in auth layer (PostgreSQL responsibility)

---

### Alembic (SQLAlchemy)

- **Description:** Python SQLAlchemy migration tool with autogenerate capability that diffs ORM models against live databases; Python-ecosystem only, integrates deeply with SQLAlchemy ORM.
- **API Documentation:** https://alembic.sqlalchemy.org/ (Official) | https://alembic.sqlalchemy.org/en/latest/api/overview.html (API Reference)
- **SDKs/Libraries:**
  - Python Package: https://pypi.org/project/alembic/
  - SQLAlchemy Integration: Built-in, works with SQLAlchemy 1.4+
  - CLI: Command-line tool (`alembic` command)
  - GitHub Repository: https://github.com/sqlalchemy/alembic
- **Developer Guide:** https://alembic.sqlalchemy.org/en/latest/tutorial.html
- **Standards:** Python-native (no polyglot support), SQL generation via SQLAlchemy, semver-inspired migration versioning
- **Authentication:** Database credentials passed via SQLAlchemy connection strings

---

### SchemaHero

- **Description:** Kubernetes operator for declarative database schema management; expresses table schemas as Kubernetes Custom Resources (CRDs); GitOps-native for Kubernetes environments.
- **API Documentation:** https://github.com/schemahero/schemahero (GitHub Repo) | https://schemahero.io (Official Site — under construction as of research date)
- **SDKs/Libraries:**
  - Kubernetes Operator: CRD-based (kubectl apply compatible)
  - YAML/Manifests: Kubernetes Custom Resources (Table, Migration CRDs)
  - kubectl: Native Kubernetes CLI (`kubectl apply`, `kubectl describe migration`)
  - ArgoCD/Flux: GitOps-native workflows via declarative YAML sync
- **Developer Guide:** https://github.com/schemahero/schemahero/docs/ and community documentation
- **Standards:** Kubernetes Custom Resource Definitions (CRDs), OpenAPI v3 schema validation for CRDs, GitOps (Flux/ArgoCD compatible), Semantic Versioning for operator releases
- **Authentication:** Kubernetes RBAC for migration approval workflows; database authentication via secrets

---

## Notes

### Gaps in Current Standards

1. **Database Migration API Standardization:** No formal ISO/IETF standard exists specifically for database migration APIs. The field relies on OpenAPI 3.1 for REST documentation and gRPC for performance-critical APIs, but lacks a unified schema definition language for changesets.

2. **Drift Detection Metrics:** While DORA metrics address change failure rate, no industry standard quantifies "drift" severity or "schema divergence" impact. Bytebase and Atlas provide proprietary metrics; formalization would benefit the ecosystem.

3. **AI-Assisted Migration Safety:** Emerging standards around AI-generated code (e.g., NIST AI RMF) do not yet address database migration safety analysis. This represents an open opportunity for the migration manager project.

### Emerging Standards & Future Directions

1. **NIST AI Risk Management Framework (AI RMF):** May influence future requirements for AI-powered migration analysis (Atlas Copilot approach).

2. **JSON Schema 2024-12:** Latest JSON Schema iteration may enable richer semantic validation for changesets and schema definitions.

3. **OIDC for Database Authorization:** Emerging trends toward OIDC-bound database credentials (e.g., AWS RDS IAM auth) will require standardization in migration tools.

4. **Kubernetes-Native Standards:** As SchemaHero demonstrates, K8s operator patterns are becoming de facto standard for cloud-native migration tooling; expect CNCF formalization.

### Compliance & Regulatory Alignment

- **Regulated Industries (Finance, Healthcare):** SOC 2 Type II and ISO 27001 audit trail requirements are table-stakes for enterprise adoption. Bytebase and Liquibase Secure address this; consider as MVP must-haves.
- **Data Residency & GDPR:** Schema migration tools must support multi-region deployment and audit logging per GDPR Article 32. Bytebase's multi-database support aligns well here.

### Recommended Standard Adoption Path

1. **Must-have:** OpenAPI 3.1 for API documentation, OAuth 2.0 / OIDC for authentication, Semantic Versioning for changeset ordering.
2. **Should-have:** DORA metrics integration, ISO 27001 audit trail compliance, GitOps principles (OpenGitOps v1.0).
3. **Nice-to-have:** Kubernetes CRD integration (SchemaHero pattern), gRPC for high-performance APIs, JSON Schema 2024-12 for advanced validation.
