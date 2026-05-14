# SQL Curation Rules

## Idiomatic Patterns

What senior SQL engineers write:

- **CTEs for readability**: `WITH active_users AS (SELECT ...) SELECT ...` instead of nested subqueries. CTEs name intermediate results, making the query read top-to-bottom like prose.
- **Explicit column lists**: `SELECT id, name, email` — never `SELECT *`. Wildcards break when columns are added, reordered, or renamed, and pull unnecessary data over the wire.
- **`EXISTS` over `IN` for correlated checks**: `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)` short-circuits at the first match; `IN` materializes the full subquery result.
- **Window functions over self-joins**: `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC)` eliminates the duplicate-eliminating self-join and runs in a single pass.
- **`COALESCE` over `CASE WHEN IS NULL`**: `COALESCE(discount, 0)` is shorter, intent-clear, and supported universally. Reserve `CASE` for multi-branch conditional logic.
- **Parameterized queries always**: Use `$1`/`?`/`@param` placeholders. Never build SQL by string interpolation — it is an injection vector and prevents query plan caching.
- **Meaningful aliases**: `FROM users u JOIN orders o ON o.user_id = u.id`. Single-letter aliases that collide across a large query (`a`, `b`) make maintenance harder, not easier.
- **`UNION ALL` over `UNION` when no duplicates**: `UNION` performs a full deduplication pass. If the sets are already disjoint, `UNION ALL` avoids the sort/hash step.
- **Consistent casing**: UPPERCASE SQL keywords (`SELECT`, `FROM`, `WHERE`, `JOIN`), lowercase table and column names. Consistency aids readability and diff review.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in SQL:

- **`SELECT *` everywhere**: Wildcard selects mask schema dependencies, drag unused columns across the network, and break typed result-set mapping when the schema evolves.
- **N+1 query patterns**: Fetching a list of records in one query, then issuing a separate query per row in application code. One JOIN or a `WHERE id = ANY($1)` batch replaces hundreds of round trips.
- **Nested subqueries 3+ levels deep**: Deeply nested `SELECT (SELECT (SELECT ...))` structures are unreadable and often less efficient than equivalent CTEs that the optimizer can materialize once.
- **`LIKE '%value%'` (leading wildcard)**: A leading `%` prevents the planner from using a B-tree index, resulting in a full sequential scan. Use full-text search (`tsvector`/`MATCH AGAINST`) or a trigram index instead.
- **Implicit joins**: `FROM users u, orders o WHERE u.id = o.user_id` — the old comma-join syntax obscures intent, makes outer joins awkward, and is easy to accidentally cross-join.
- **`ORDER BY` without `LIMIT` on large results**: Sorting an unbounded result set forces a full sort before the first row is returned. Add `LIMIT`/`FETCH FIRST` unless the caller truly needs every row sorted.
- **`DISTINCT` as a fix for bad join duplicates**: `SELECT DISTINCT` to paper over a fanout from a mis-specified join hides the real bug and adds a deduplication pass. Fix the join cardinality instead.
- **Redundant `WHERE 1=1`**: A no-op condition added to make dynamic query building easier. Use a proper query builder or parameterized list instead.
- **String concatenation in SQL**: Building query strings by concatenating user input opens SQL injection vulnerabilities. Always use parameterized queries or a safe query builder.
- **Missing `NOT NULL` constraints**: Allowing nullable columns when the domain guarantees a value forces every consumer to handle `NULL` and causes silent aggregation bugs (`SUM`, `COUNT`, `AVG` all ignore `NULL`s).

## Simplification Strategies

### 1. Replace nested subqueries with CTEs

Before — three-level nesting to compute billable order totals per active billing period:

```sql
SELECT
    bp.id,
    bp.user_id,
    sub.total
FROM billing_periods bp
JOIN (
    SELECT
        o.billing_period_id,
        SUM(li.amount) AS total
    FROM orders o
    JOIN (
        SELECT order_id, SUM(unit_price * quantity) AS amount
        FROM line_items
        WHERE cancelled = FALSE
        GROUP BY order_id
    ) li ON li.order_id = o.id
    WHERE o.status = 'completed'
    GROUP BY o.billing_period_id
) sub ON sub.billing_period_id = bp.id
WHERE bp.active = TRUE;
```

After — named CTEs that build on each other:

```sql
WITH line_totals AS (
    SELECT order_id, SUM(unit_price * quantity) AS amount
    FROM line_items
    WHERE cancelled = FALSE
    GROUP BY order_id
),
order_totals AS (
    SELECT o.billing_period_id, SUM(lt.amount) AS total
    FROM orders o
    JOIN line_totals lt ON lt.order_id = o.id
    WHERE o.status = 'completed'
    GROUP BY o.billing_period_id
)
SELECT bp.id, bp.user_id, ot.total
FROM billing_periods bp
JOIN order_totals ot ON ot.billing_period_id = bp.id
WHERE bp.active = TRUE;
```

### 2. Replace `SELECT *` with explicit columns

Before — wildcard select leaks internal columns and breaks typed mapping:

```sql
SELECT *
FROM users u
JOIN user_profiles p ON p.user_id = u.id
WHERE u.active = TRUE;
```

After — explicit columns with unambiguous aliases:

```sql
SELECT
    u.id,
    u.email,
    u.created_at,
    p.display_name,
    p.avatar_url
FROM users u
JOIN user_profiles p ON p.user_id = u.id
WHERE u.active = TRUE;
```

### 3. Replace N+1 pattern with a JOIN

Before — list fetch in SQL followed by per-item query in application code:

```sql
-- Query 1: fetch orders
SELECT id, user_id, total FROM orders WHERE status = 'pending';

-- Then in application code, for each order:
SELECT name, email FROM users WHERE id = ?;  -- repeated N times
```

After — single query with a JOIN returns everything in one round trip:

```sql
SELECT
    o.id,
    o.total,
    u.name,
    u.email
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending';
```

## Dead Code Signals

SQL-specific indicators that schema objects or queries are no longer exercised:

- **Unused stored procedures/functions**: Procedures or functions that are defined in migrations but never called from application code, other procedures, or scheduled jobs. Search the codebase for the routine name before removing.
- **Unused views**: Views that appear in no application query, no dependent view, and no reporting tool. Check `pg_stat_user_tables` (or equivalent) for zero-scan views.
- **Dead migration files**: A migration that creates a table or column followed by a subsequent migration that drops it. The net effect is nothing; both migrations are noise that slows cold-start replay.
- **Commented-out columns in `CREATE TABLE`**: Columns left in a `-- comment` inside a DDL statement that were disabled mid-development but never cleaned up from the migration.
- **Unused indexes**: Indexes that the query planner never selects. In PostgreSQL, `pg_stat_user_indexes.idx_scan = 0` (after a representative workload) identifies candidates for removal. In MySQL, `sys.schema_unused_indexes` provides the same signal.
- **Orphaned foreign keys**: Foreign key constraints referencing tables or columns that have been logically deprecated but not yet dropped. These enforce referential integrity for relationships that application code no longer exercises.
