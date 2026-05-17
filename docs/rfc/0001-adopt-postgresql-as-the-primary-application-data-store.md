---
id: RFC-0001
title: "Adopt PostgreSQL as the primary application data store"
status: Accepted
author: Donald Gifford
created: 2026-05-17
---
<!-- markdownlint-disable-file MD025 MD041 -->

# RFC 0001: Adopt PostgreSQL as the primary application data store

**Status:** Accepted
**Author:** Donald Gifford
**Date:** 2026-05-17

<!--toc:start-->
- [Summary](#summary)
- [Problem Statement](#problem-statement)
- [Proposed Solution](#proposed-solution)
- [Design](#design)
  - [Cluster topology](#cluster-topology)
  - [Connection management](#connection-management)
  - [Schema migration tooling](#schema-migration-tooling)
- [Alternatives Considered](#alternatives-considered)
- [Implementation Phases](#implementation-phases)
  - [Phase 1: Greenfield + auth + catalog (Q3 2026)](#phase-1-greenfield--auth--catalog-q3-2026)
  - [Phase 2: Monolith decomposition (Q4 2026 – Q1 2027)](#phase-2-monolith-decomposition-q4-2026--q1-2027)
  - [Phase 3: Cutover + decommission (Q2 2027)](#phase-3-cutover--decommission-q2-2027)
- [Risks and Mitigations](#risks-and-mitigations)
- [Success Criteria](#success-criteria)
- [References](#references)
<!--toc:end-->

## Summary

Adopt **PostgreSQL 16** as the system of record for all application services. New services start on Postgres; existing MySQL and document-store deployments migrate per the phased plan below. Postgres becomes the only datastore on the architectural happy path; specialized stores (Redis, S3, search indexes) remain for caching and large-blob/full-text use cases.

## Problem Statement

We currently operate three primary datastores across the platform:

- **MySQL 5.7** behind the legacy monolith (12 services, ~3 TB)
- **MongoDB 4.4** for product catalog + content metadata (~600 GB)
- **DynamoDB** for two newer services (low volume, ~40 GB)

The fragmentation costs us:

1. **Operational tax.** Three on-call runbooks, three backup strategies, three restore drills, three different `EXPLAIN` dialects.
2. **Data integrity gaps.** MongoDB has no foreign keys; the content team has shipped four `BAD_REFERENCE` outages in the last 18 months when application-layer joins broke.
3. **Stale tooling.** MySQL 5.7 reached end-of-life in October 2023; we're paying for extended support.
4. **Skill drift.** Hiring + interviewing for "general SQL" is easier than the union of all three.

> [!NOTE]
> This RFC is the substrate for the [event-driven architecture proposal](./0006-adopt-an-event-driven-architecture-using-nats-jetstream.md) — outbox-pattern publishing assumes a transactional primary store.

## Proposed Solution

Single primary: **PostgreSQL 16** on managed RDS (Aurora-compatible).

Why Postgres specifically:

- **JSONB** covers the document use cases that pushed us to MongoDB originally.
- **Logical replication** + the **outbox pattern** unlock event-driven workflows without dual-writes.
- **Row-level security** lets us tighten the multi-tenant surface without service-layer hacks.
- **`pgvector`** opens the door to embedding-backed search without a new store.

## Design

### Cluster topology

| Environment | Instance | Storage | Replicas | Backup |
|-------------|----------|---------|----------|--------|
| Production  | `db.r7g.4xlarge` | 4 TB gp3 | 2 read | PITR 14d |
| Staging     | `db.r7g.xlarge`  | 1 TB gp3 | 1 read | PITR 7d  |
| Dev         | `db.t4g.large`   | 200 GB   | none    | snapshot daily |

### Connection management

All services connect through **PgBouncer** in transaction-pooling mode. Per-service `max_pool_size = ceil(replicas * 1.5)`; the application connection pool stays small (`HikariCP max=10`).

```sql
-- Per-service role pattern
CREATE ROLE svc_catalog WITH LOGIN PASSWORD :pw NOINHERIT;
GRANT CONNECT ON DATABASE app TO svc_catalog;
GRANT USAGE ON SCHEMA catalog TO svc_catalog;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA catalog TO svc_catalog;
ALTER DEFAULT PRIVILEGES IN SCHEMA catalog
  GRANT SELECT, INSERT, UPDATE ON TABLES TO svc_catalog;
```

### Schema migration tooling

Standardize on **golang-migrate** for Go services and **Flyway** for JVM services. Migrations live next to the service code; CI fails the build if a migration touches another service's schema.

## Alternatives Considered

- **CockroachDB.** Operationally heavier; we don't have the multi-region scale to justify the cost premium today.
- **MySQL 8.** Closes the version gap but doesn't help with the document/relational split. JSON support is real but weaker than Postgres JSONB indexing.
- **Stay on three stores.** Status quo. Rejected — see *Problem Statement*.

## Implementation Phases

### Phase 1: Greenfield + auth + catalog (Q3 2026)

- Stand up production cluster.
- Migrate `auth-service` (DynamoDB → Postgres) — low volume, smallest blast radius.
- Migrate `catalog-service` (MongoDB → Postgres with JSONB columns for variant attributes).

### Phase 2: Monolith decomposition (Q4 2026 – Q1 2027)

- Migrate domain by domain out of the MySQL monolith. Each domain ships with its own schema, foreign-key boundary, and outbox table.

### Phase 3: Cutover + decommission (Q2 2027)

- Final monolith domain cut. MySQL cluster moved to read-only for 30 days, then decommissioned.
- MongoDB cluster decommissioned after Phase 1 completes.

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Migration data loss | High | Low | Dual-write window + checksums per table |
| Connection exhaustion | High | Medium | PgBouncer + per-service connection budgets |
| JSONB performance vs Mongo | Medium | Medium | GIN indexes + benchmark with production-shape data |
| Skill ramp on Postgres-specific features | Low | High | Internal training + paired runbooks |

## Success Criteria

- 100% of services on Postgres by end of Q2 2027.
- Operational on-call alerts attributable to datastore tier drop by 50% YoY.
- Zero data loss incidents during the migration window.

## References

- [PostgreSQL 16 release notes](https://www.postgresql.org/docs/16/release-16.html)
- [Outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- Related: [RFC-0006](./0006-adopt-an-event-driven-architecture-using-nats-jetstream.md), [RFC-0003](./0003-standardize-on-opentelemetry-for-traces-metrics-and-logs.md)
