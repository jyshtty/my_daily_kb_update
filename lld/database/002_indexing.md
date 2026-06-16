# Database Indexing

## What is an Index?
A data structure that speeds up reads by letting the DB find rows without a full table scan.
Trade-off: faster reads, slower writes, more storage.

```
Without index: SELECT * FROM users WHERE email = 'x@y.com'
               → scan all 10M rows one by one (O(n))

With index:    → B-tree lookup → O(log n) — find in microseconds
```

## How B-Tree Index Works (Default in PostgreSQL, MySQL)

Balanced tree where leaf nodes hold actual row pointers.
```
           [M]
          /   \
       [G]     [T]
      /   \   /   \
    [A-F] [H-L] [N-S] [U-Z]
```
- Supports: =, <, >, BETWEEN, ORDER BY, LIKE 'prefix%'
- Does NOT help: LIKE '%suffix', functions on column (LOWER(name))

## Index Types

### B-Tree (Default)
```sql
CREATE INDEX idx_users_email ON users(email);
```
Good for: equality and range queries on most data types.

### Hash Index
```sql
CREATE INDEX idx_users_email ON users USING HASH(email);
```
- Faster than B-tree for exact equality (=)
- Useless for range queries (<, >, BETWEEN)
- PostgreSQL only; not crash-safe in older versions

### Composite Index
```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```
- Covers queries on (user_id), (user_id, status)
- Does NOT cover queries on (status) alone
- **Left-prefix rule**: index (a, b, c) helps queries on a, (a,b), (a,b,c) — not b or c alone

### Partial Index
Index only a subset of rows.
```sql
-- Only index active users — smaller, faster
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

### Covering Index
Include all columns a query needs so it never touches the main table.
```sql
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (status, created_at);
-- Query hits index only — no table lookup
SELECT status, created_at FROM orders WHERE user_id = 123;
```

### Full-Text Index
```sql
CREATE INDEX idx_posts_fts ON posts USING GIN(to_tsvector('english', body));
SELECT * FROM posts WHERE to_tsvector('english', body) @@ to_tsquery('python & async');
```

## When Indexes Help vs Hurt

| Situation | Index? |
|---|---|
| High-cardinality column (email, user_id) | Yes |
| Low-cardinality column (status: 3 values) | Usually no — full scan faster |
| Frequently filtered/joined column | Yes |
| Column only used in SELECT (not WHERE) | No |
| Write-heavy table (logs, events) | Careful — every write updates index |
| Small table (<10k rows) | Usually no — full scan is fine |

## Query Planner — EXPLAIN
Always check if your index is actually used:
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'x@y.com';

-- Good: Index Scan using idx_users_email
-- Bad:  Seq Scan (index not used — check why)
```

Common reasons index is ignored:
- Column has a function applied: `WHERE LOWER(email) = ...` → use functional index
- Low cardinality: DB decides full scan is cheaper
- Outdated statistics: run `ANALYZE table_name`

## Index Maintenance
- Indexes bloat over time (dead rows from updates/deletes)
- Rebuild periodically:
```sql
REINDEX INDEX idx_users_email;        -- locks table
REINDEX INDEX CONCURRENTLY idx_users_email;  -- no lock (Postgres 12+)
```

## Multi-Column Index Column Order Rule
Put the most selective column first, OR the column used in WHERE most often.
```sql
-- Query: WHERE user_id = ? AND created_at > ?
CREATE INDEX ON orders(user_id, created_at);  -- user_id first (higher cardinality)
```

## Key Rules
1. Index foreign keys — joins without index = full scan on every join
2. One index per frequent query pattern, not one per column
3. Too many indexes = slow writes — audit unused indexes regularly
4. Composite index column order matters — follow the left-prefix rule
