# Review: Problems & Improvements

## Bugs / Inconsistencies

### 1. `having` missing from `QueryDefinition`

`SqlParts` has `having: WhereCondition[]` and test scenario #22 tests HAVING, but `QueryDefinition` had no `having` field.

**Resolution:** Added `having?: QueryFilter[]` to `QueryDefinition`. Reuses existing `QueryFilter` type — semantically same as WHERE but applied post-GROUP BY.

### 2. No join type in `QueryJoin`

`JoinClause` in `SqlParts` has `type: 'inner' | 'left'`, but `QueryJoin` had no way to specify it.

**Resolution:** Added `type?: 'inner' | 'left'` to `QueryJoin`, defaulting to `'left'` (safe for nullable FKs).

### 3. Debug log tag mismatch

Debug example used `[access]`, but `DebugLogEntry.phase` defines it as `'access-control'`.

**Resolution:** Changed example to `[access-control]`.

### 4. Missing `orders → products` relation

Test scenario #2 ("Join within same PG: orders + products") requires a relation, but `ordersRelations` only had `customerId → users` and `tenantId → tenants`.

**Resolution:** Added `productId` column to orders table + `productId → products.id` relation.

### 5. "Matches eng project ecosystem"

Design Decisions table had an internal reference.

**Resolution:** Changed to "Type-safe, wide ecosystem".

---

## Missing Test Fixtures

### 6. No column definitions for 3 tables

`metrics`, `invoices`, and `ordersArchive` were used in test scenarios and roles but had no sample column definitions.

**Resolution:** Added full column definitions for all three tables.

---

## Unaddressed Edge Cases

### 7. `byIds` with composite primary keys

`primaryKey: string[]` can be multi-column, but `byIds?: (string | number)[]` is a flat array.

**Resolution:** Restricted `byIds` to single-column primary keys (documented in the field comment). Composite PK lookups use filters instead.

### 8. `byIds` + filters

If a `byIds` query also has `filters`, should cache results be post-filtered?

**Resolution:** Cache strategy is skipped when `filters` are present alongside `byIds`. Documented in Priority 0 cache section.

### 9. Cache + masking

Cache returns pre-fetched data — is masking still applied?

**Resolution:** Added explicit note: masking is applied on cache hits identically to DB results.

### 10. `count` mode + `columns`

When `executeMode = 'count'`, are `columns` ignored?

**Resolution:** Added clarification: in count mode, `columns`, `orderBy`, `limit`, and `offset` are ignored.

---

## Minor Type Improvements

### 11. `orderBy` inline type

`orderBy` used an inline `{ column: string; direction: 'asc' | 'desc' }[]` while other clauses had named interfaces.

**Resolution:** Extracted to `QueryOrderBy` interface for consistency.

### 12. `WhereCondition.operator` is `string`

`QueryFilter.operator` uses a proper union type, `WhereCondition.operator` in `SqlParts` is just `string`.

**Resolution:** Kept as `string` — it's internal, and dialects may emit operators not in the public union (e.g. `ILIKE`, `ANY`). Added a comment explaining why.

### 13. Return type of `createMultiDb()` not defined

The init example calls `createMultiDb({...})` but the returned object's type wasn't documented.

**Resolution:** Added `MultiDb` interface with `query()` method signature.

---

## Structural

### 14. Objective / Overview overlap

Both sections mentioned strategy selection, access control, name mapping, debug logging, and execution modes.

**Resolution:** Merged into a single "Objective" section. The Overview was redundant — the architecture diagram already covers "how".
