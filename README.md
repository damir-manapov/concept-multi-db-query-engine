# @mkven/multi-db — Concept Document

## Overview

A metadata-driven query planner and executor that abstracts over heterogeneous database backends. Converts a declarative query definition into optimized database requests with:

- Automatic strategy selection (direct, cached, materialized replica, Trino cross-db)
- Role-based access control (column-level visibility + row-level filters)
- Name mapping between API-facing names and physical database names
- Structured debug logging for full pipeline transparency
- Support for SQL-only or SQL+execution modes

## Target Databases

| Engine | Role |
|---|---|
| **Postgres** | Primary OLTP, supports native RLS |
| **ClickHouse** | Analytics, columnar storage |
| **Iceberg** (via Trino) | Archival / lakehouse |
| **Trino** | Cross-database query federation |
| **Redis** | Cache layer for by-ID lookups |

Tables may live in any database. Some tables are replicated between databases via Debezium (CDC). The planner uses this topology to pick the optimal execution path.

---

## Architecture

```
┌─────────────────────────────────────┐
│       Query Definition (DSL)        │
│  { from, columns, filters, joins }  │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  1. VALIDATION                      │
│  - table/column existence           │
│  - role permissions                 │
│  - filter operator validity         │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  2. RLS INJECTION                   │
│  - per-role mandatory filters       │
│  - column trimming                  │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  3. QUERY PLANNING                  │
│  Strategy selection:                │
│  P0: Cache (redis, by-ID only)      │
│  P1: Single-DB direct               │
│  P2: Materialized replica           │
│  P3: Trino cross-DB                 │
│  P4: Error (unreachable)            │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  4. NAME RESOLUTION                 │
│  apiName → physicalName mapping     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  5. SQL GENERATION                  │
│  Dialect-specific SQL output        │
│  (Postgres / ClickHouse / Trino)    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  6. EXECUTION (optional)            │
│  Run against correct connection     │
│  Return typed results               │
└─────────────────────────────────────┘
```

Every phase emits structured debug log entries.

---

## Metadata Model

### Databases

```ts
type DatabaseEngine = 'postgres' | 'clickhouse' | 'iceberg'

interface DatabaseMeta {
  id: string                          // logical id: 'main-pg', 'analytics-ch'
  engine: DatabaseEngine
  trinoCatalog?: string               // if accessible via trino, its catalog name
}
```

### Tables

```ts
interface TableMeta {
  id: string                          // logical table id: 'orders'
  apiName: string                     // exposed to API consumers: 'orders'
  database: string                    // references DatabaseMeta.id
  physicalName: string                // actual table name in DB: 'public.orders'
  columns: ColumnMeta[]
  primaryKey: string[]                // using apiName references
  relations: RelationMeta[]
}
```

### Columns

```ts
interface ColumnMeta {
  apiName: string                     // 'customerEmail'
  physicalName: string                // 'customer_email'
  type: string                        // logical type: 'string', 'int', 'date', 'uuid', 'decimal', 'timestamp', etc.
  nullable: boolean
  indexed?: boolean                   // hint for planner (prefer indexed filters)
}
```

### Relations

```ts
interface RelationMeta {
  column: string                      // apiName of FK column in this table
  references: { table: string; column: string }  // apiName references
  type: 'many-to-one' | 'one-to-many' | 'one-to-one'
}
```

### External Sync (Debezium CDC)

```ts
interface ExternalSync {
  sourceTable: string                 // TableMeta.id of the original
  targetDatabase: string              // DatabaseMeta.id where materialized
  targetPhysicalName: string          // physical name in target DB
  method: 'debezium'
  estimatedLag: 'realtime' | 'seconds' | 'minutes' | 'hours'
}
```

### Cache

```ts
interface CacheMeta {
  id: string
  engine: 'redis'                     // extensible later
  tables: CachedTableMeta[]
}

interface CachedTableMeta {
  tableId: string                     // references TableMeta.id
  keyPattern: string                  // e.g. 'users:{id}' or 'users:{id}:{tenantId}'
  ttl?: number                        // seconds, undefined = no expiry
  columns?: string[]                  // subset cached (apiNames); undefined = all
}
```

### Roles & Access Control

RLS is defined **per role**, not per table. Each role declares which tables it can access, which columns are visible, and what mandatory filters are applied.

```ts
interface RoleMeta {
  id: string                          // 'admin', 'manager', 'tenant-user'
  tables: TableRoleAccess[]
}

interface TableRoleAccess {
  tableId: string
  allowedColumns: string[] | '*'      // apiNames, or '*' for all
  filters?: RlsFilter[]              // mandatory WHERE conditions for this role
}

interface RlsFilter {
  column: string                      // apiName
  operator: '=' | 'in' | '!=' | 'is_null' | 'is_not_null'
  valueFrom: 'context'               // resolved from execution context
  contextKey: string                  // e.g. 'tenantId', 'userId', 'regionIds'
}
```

### Full Configuration

```ts
interface MultiDbConfig {
  databases: DatabaseMeta[]
  tables: TableMeta[]
  caches: CacheMeta[]
  externalSyncs: ExternalSync[]
  roles: RoleMeta[]
  trino?: { enabled: boolean }
}
```

Metadata source is **abstracted** — hardcoded for now, in future loaded from a database or external service and cached.

---

## Query Definition

### Query Input

```ts
interface QueryDefinition {
  from: string                        // table apiName
  columns?: string[]                  // apiNames; undefined = all allowed for role
  filters?: QueryFilter[]
  joins?: QueryJoin[]
  limit?: number
  offset?: number
  orderBy?: { column: string; direction: 'asc' | 'desc' }[]
  freshness?: 'realtime' | 'seconds' | 'minutes' | 'hours'  // acceptable lag
  byIds?: (string | number)[]        // shortcut: fetch by primary key(s)
  executeMode?: 'sql-only' | 'execute'  // default: 'execute'
}

interface QueryJoin {
  table: string                       // related table apiName
  columns?: string[]                  // columns to select from joined table
  filters?: QueryFilter[]            // filters on joined table
}

interface QueryFilter {
  column: string                      // apiName
  operator: '=' | '!=' | '>' | '<' | '>=' | '<=' | 'in' | 'like' | 'is_null' | 'is_not_null'
  value?: unknown
}
```

### Execution Context

```ts
interface ExecutionContext {
  role: string                        // RoleMeta.id
  contextValues: Record<string, unknown>  // e.g. { tenantId: 'abc', userId: '123', regionIds: ['us-east', 'us-west'] }
}
```

### Query Result

```ts
interface QueryResult<T = unknown> {
  data?: T[]                          // present if executeMode = 'execute'
  sql: string                        // always present
  params: unknown[]                  // bound parameters
  dialect: DatabaseEngine | 'trino'
  targetDatabase: string             // which DB it went to
  strategy: 'direct' | 'cache' | 'materialized' | 'trino-cross-db'
  debugLog: DebugLogEntry[]          // always collected
}

interface DebugLogEntry {
  timestamp: number
  phase: 'validation' | 'rls' | 'planning' | 'name-resolution' | 'sql-generation' | 'cache' | 'execution'
  message: string
  details?: unknown                  // structured data for inspection
}
```

---

## Query Planner Strategy

Given a query touching tables T1, T2, ... Tn:

### Priority 0 — Cache (redis)
- Only for `byIds` queries
- Check if the table has a cache config
- If cache hit for all IDs → return from cache (trim columns if needed)
- If partial hit → return cached + fetch missing from DB, merge

### Priority 1 — Single Database Direct
- ALL required tables exist in ONE database (original data)
- Generate native SQL for that engine
- This is always preferred when possible (freshest data, no overhead)

### Priority 2 — Materialized Replica
- Some tables are in different databases, BUT debezium replicas exist such that all needed data is available in one database
- Check freshness: if query requires `realtime` but replica lag is `minutes` → skip this strategy
- Always prefer the **original** database if the original data is there; use replicas for the "foreign" tables
- If multiple databases could serve via replicas, prefer the one with the most original tables

### Priority 3 — Trino Cross-Database
- Trino is enabled and all databases are registered as trino catalogs
- Generate Trino SQL with cross-catalog references
- Fallback when no single-DB path exists

### Priority 4 — Error
- Cannot reach all tables via any strategy
- Return clear error: which tables are unreachable, what's missing (trino disabled? no catalog? no sync?)

---

## Validation Rules

1. **Table existence** — only tables defined in metadata can be queried
2. **Column existence** — only columns defined in table metadata can be referenced
3. **Role permission** — if a table is not in the role's `tables` list → access denied
4. **Column permission** — if `allowedColumns` is a list and requested column is not in it → denied; if columns not specified in query, return only allowed ones
5. **Filter validity** — filter operators must be valid for the column type
6. **Join validity** — joined tables must have a defined relation in metadata
7. **Context completeness** — if role requires `contextKey: 'tenantId'`, it must be present in `ExecutionContext.contextValues`

All validation errors are descriptive and include what was expected vs what was provided.

---

## Name Mapping

Every table and column has:
- `apiName` — what the user provides in queries and receives in results
- `physicalName` — what's actually in the database

The system:
1. Accepts queries using `apiName`
2. Resolves to `physicalName` for SQL generation
3. Returns results mapped back to `apiName`

This decouples the API contract from database schema evolution.

---

## SQL Dialect Differences

| Feature | Postgres | ClickHouse | Trino |
|---|---|---|---|
| Identifier quoting | `"column"` | `` `column` `` | `"column"` |
| Parameter binding | `$1, $2` | `{p1:Type}` | `?` |
| Arrays | `= ANY($1)` | `arrayJoin` | `UNNEST` |
| Date functions | `date_trunc(...)` | `toStartOfDay(...)` | `date_trunc(...)` |
| LIMIT/OFFSET | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` |
| Boolean | `true/false` | `1/0` | `true/false` |

Each engine gets a `SqlDialect` implementation.

---

## Debug Logging — Example Flow

```
[validation]  Resolving table 'orders' → found in metadata (database: pg-main)
[validation]  Column 'id' → valid (type: uuid, indexed: true)
[validation]  Column 'total' → valid (type: decimal)
[validation]  Column 'internalNote' → DENIED for role 'tenant-user' (not in allowedColumns)
[rls]         Role 'tenant-user' requires filter: orders.tenantId = context.tenantId
[rls]         Injecting WHERE tenant_id = $1 (resolved: 'abc-123')
[rls]         Trimming columns to allowed set: [id, total, status, createdAt]
[planning]    Tables needed: [orders(pg-main), customers(pg-main)]
[planning]    All tables in 'pg-main' → strategy: DIRECT
[name-res]    orders.total → public.orders.total_amount
[name-res]    orders.tenantId → public.orders.tenant_id
[sql-gen]     Dialect: postgres
[sql-gen]     SELECT "id", "total_amount" AS "total", ... FROM "public"."orders" WHERE "tenant_id" = $1
[execution]   Executing on pg-main → 42 rows, 8ms
```

For cross-database scenario:
```
[planning]    Tables needed: [orders(pg-main), events(ch-analytics)]
[planning]    Not in same database. Checking materializations...
[planning]    Found: orders → ch-analytics (debezium, lag: seconds)
[planning]    Query freshness: 'minutes' → materialized replica acceptable
[planning]    Strategy: MATERIALIZED via ch-analytics
[planning]      orders → default.orders_replica (materialized)
[planning]      events → default.events (original)
[sql-gen]     Dialect: clickhouse
```

---

## Recommended Test Configuration

### Databases

| ID | Engine | Trino Catalog | Purpose |
|---|---|---|---|
| `pg-main` | postgres | `pg_main` | Primary OLTP |
| `pg-tenant` | postgres | `pg_tenant` | Tenant-specific data |
| `ch-analytics` | clickhouse | `ch_analytics` | Analytics / events |
| `iceberg-archive` | iceberg | `iceberg_archive` | Historical archives |

### Tables

| Table ID | API Name | Database | Physical Name |
|---|---|---|---|
| `users` | users | pg-main | public.users |
| `orders` | orders | pg-main | public.orders |
| `products` | products | pg-main | public.products |
| `tenants` | tenants | pg-tenant | public.tenants |
| `invoices` | invoices | pg-tenant | public.invoices |
| `events` | events | ch-analytics | default.events |
| `metrics` | metrics | ch-analytics | default.metrics |
| `orders-archive` | ordersArchive | iceberg-archive | warehouse.orders_archive |

### External Syncs (Debezium)

| Source | Target DB | Target Physical Name | Lag |
|---|---|---|---|
| orders | ch-analytics | default.orders_replica | seconds |
| orders | iceberg-archive | warehouse.orders_current | minutes |
| tenants | pg-main | replicas.tenants | seconds |

### Cache (Redis)

| Table | Key Pattern | TTL |
|---|---|---|
| users | `users:{id}` | 300s |
| products | `products:{id}` | 600s |

### Roles

| Role | Access |
|---|---|
| `admin` | All tables, all columns, no filters |
| `tenant-user` | orders + customers, subset columns, forced `tenantId = context.tenantId` |
| `regional-manager` | orders (all cols), forced `regionId IN context.regionIds` |
| `analytics-reader` | events + metrics + ordersArchive only |
| `no-access` | No tables (edge case) |

### Test Scenarios

| # | Scenario | Tables | Expected Strategy |
|---|---|---|---|
| 1 | Single PG table | orders | direct → pg-main |
| 2 | Join within same PG | orders + products | direct → pg-main |
| 3 | Cross-PG, debezium available | orders + tenants | materialized → pg-main (tenants replicated) |
| 4 | Cross-PG, no debezium | orders + invoices | trino cross-db |
| 5 | PG + CH, debezium available | orders + events | materialized → ch-analytics (orders replicated) |
| 6 | PG + CH, no debezium | products + events | trino cross-db |
| 7 | PG + Iceberg | orders + ordersArchive | trino or materialized |
| 8 | By-ID with cache hit | users byIds=[1,2,3] | cache → redis |
| 9 | By-ID cache miss | orders byIds=[1,2] | direct → pg-main |
| 10 | By-ID partial cache | users byIds=[1,2,3] (1,2 cached) | cache + direct merge |
| 11 | Freshness=realtime, has materialized | orders + events (realtime) | skip materialized (lag=seconds), trino |
| 12 | Freshness=hours, has materialized | orders + events (hours ok) | materialized → ch-analytics |
| 13 | Admin role | any table | all columns, no RLS filters |
| 14 | Tenant-user role | orders | subset columns + forced WHERE tenantId=X |
| 15 | No-access role | any table | denied |
| 16 | Column trimming on byIds | users byIds + limited columns in role | only intersected columns |
| 17 | Invalid table name | nonexistent | validation error |
| 18 | Invalid column name | orders.nonexistent | validation error |
| 19 | Trino disabled | cross-db query | error with explanation |
| 20 | Missing context value | tenant-user role, no tenantId in context | validation error |

### Sample Column Definitions (orders table)

```ts
const ordersColumns: ColumnMeta[] = [
  { apiName: 'id',           physicalName: 'id',              type: 'uuid',      nullable: false, indexed: true  },
  { apiName: 'tenantId',     physicalName: 'tenant_id',       type: 'uuid',      nullable: false, indexed: true  },
  { apiName: 'customerId',   physicalName: 'customer_id',     type: 'uuid',      nullable: false, indexed: true  },
  { apiName: 'regionId',     physicalName: 'region_id',       type: 'string',    nullable: false, indexed: true  },
  { apiName: 'total',        physicalName: 'total_amount',    type: 'decimal',   nullable: false                 },
  { apiName: 'status',       physicalName: 'order_status',    type: 'string',    nullable: false, indexed: true  },
  { apiName: 'internalNote', physicalName: 'internal_note',   type: 'string',    nullable: true                  },
  { apiName: 'createdAt',    physicalName: 'created_at',      type: 'timestamp', nullable: false                 },
]
```

---

## Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Package name | `@mkven/multi-db` | Org-scoped, reusable |
| Language | TypeScript | Matches eng project ecosystem |
| Query format | Typed object literals | Type-safe, IDE support, no parser |
| Metadata source | Abstracted (hardcoded now, DB/service later) | Flexibility without premature complexity |
| Execution mode | SQL-only or SQL+execution | Users choose per query |
| Freshness | Prefer original, allow specifying lag tolerance | Correctness by default, performance opt-in |
| RLS | Per role, not per table | Roles are the access boundary, not tables |
| Name mapping | apiName ↔ physicalName on tables and columns | Decouples API from schema |
| Cache | Redis for by-ID lookups, fallback to DB | Fast reads for hot entities |
| Debug logging | Structured entries per pipeline phase | Essential for diagnosing unexpected behavior |
| Validation | Strict — only metadata-defined entities | Fail fast with clear errors |
| Imports | Absolute paths, no `../` | Clean, refactor-friendly |

---

## Project Structure

```
@mkven/multi-db/
├── package.json
├── tsconfig.json
├── biome.json
├── vitest.config.ts
├── README.md
├── CONCEPT.md
├── src/
│   ├── index.ts                 # public API
│   ├── types/
│   │   ├── metadata.ts          # DatabaseMeta, TableMeta, ColumnMeta, etc.
│   │   ├── query.ts             # QueryDefinition, QueryFilter, QueryJoin
│   │   ├── result.ts            # QueryResult, DebugLogEntry
│   │   └── context.ts           # ExecutionContext
│   ├── metadata/
│   │   ├── registry.ts          # MetadataRegistry — stores and indexes all metadata
│   │   └── loader.ts            # abstracted loader interface
│   ├── validation/
│   │   ├── validator.ts         # query validation against metadata + role
│   │   └── errors.ts            # typed validation errors
│   ├── rls/
│   │   ├── injector.ts          # inject role-based filters + column trimming
│   │   └── resolver.ts          # resolve context values
│   ├── planner/
│   │   ├── planner.ts           # QueryPlanner — strategy selection
│   │   ├── strategies.ts        # direct, materialized, trino, cache
│   │   └── graph.ts             # database connectivity / sync graph
│   ├── dialects/
│   │   ├── dialect.ts           # SqlDialect interface
│   │   ├── postgres.ts
│   │   ├── clickhouse.ts
│   │   └── trino.ts
│   ├── generator/
│   │   ├── generator.ts         # SQL generation from resolved plan
│   │   └── fragments.ts         # reusable SQL building blocks
│   ├── cache/
│   │   ├── cache.ts             # CacheProvider interface
│   │   └── redis.ts             # Redis implementation
│   ├── executor/
│   │   ├── executor.ts          # execute against correct connection
│   │   └── connections.ts       # connection pool management
│   └── debug/
│       └── logger.ts            # DebugLogger — collects entries per query
└── tests/
    ├── fixtures/
    │   └── testConfig.ts        # the full test config described above
    ├── validation/
    ├── rls/
    ├── planner/
    ├── generator/
    ├── cache/
    └── e2e/
```

---

## Open Items for Further Discussion

- [ ] Builder pattern API on top of object literals?
- [ ] Aggregation support (GROUP BY, COUNT, SUM)?
- [ ] Subquery / nested filter support?
- [ ] Write operations (INSERT/UPDATE/DELETE) — future scope?
- [ ] Connection pool configuration and lifecycle
- [ ] Trino session properties for optimization
- [ ] Metrics / observability integration (OpenTelemetry?)
- [ ] Schema migration strategy for metadata storage
- [ ] Pagination strategy (cursor-based vs offset?)
- [ ] How to handle schema drift between original and replicated tables
