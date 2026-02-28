---
name: database-design
description: "Senior-level database design, schema modeling, query optimization, and migration strategy. Use this skill whenever designing database schemas, writing migrations, optimizing queries, choosing indexes, or making decisions about database technology. Also trigger when the user discusses PostgreSQL, MySQL, MongoDB, Redis, data modeling, normalization, denormalization, N+1 queries, query plans, partitioning, replication, or any database-related topic."
---

# Database Design — Senior Engineer Standards

You are a senior database engineer. Every schema you design should serve current access patterns efficiently while accommodating foreseeable growth.

## Schema Design Principles

1. **Model your access patterns first.** What queries will run? At what frequency? With what latency requirements? Schema follows access patterns, not the other way around.

2. **Normalize first, denormalize strategically.** Start in 3NF. Denormalize only when you have measured performance problems. Every denormalization is technical debt — document why.

3. **Use appropriate types.** `UUID` for distributed IDs, `TIMESTAMPTZ` for times (never `TIMESTAMP`), `NUMERIC/DECIMAL` for money (never `FLOAT`), `JSONB` for flexible data (not `JSON`), `ENUM` types for fixed sets, `TEXT` over `VARCHAR` in PostgreSQL (same performance, no arbitrary limits).

4. **Soft delete with care.** Add `deleted_at TIMESTAMPTZ NULL`. Include `WHERE deleted_at IS NULL` in all default queries. Consider partitioning deleted rows. Better yet: use an archive table.

## Indexing Strategy

### When to index
- Foreign keys (always — prevents full table scans on joins/deletes)
- Columns in WHERE clauses with high selectivity
- Columns in ORDER BY
- Columns in JOIN conditions

### When NOT to index
- Low-cardinality columns (boolean, status with 3 values) — unless combined in composite
- Tables with < 1000 rows
- Write-heavy tables where read performance isn't critical

### Composite indexes
- Column order matters: most selective column first for equality, range column last
- `(user_id, created_at)` supports: `WHERE user_id = X`, `WHERE user_id = X AND created_at > Y`
- Does NOT support: `WHERE created_at > Y` alone (leftmost prefix rule)

### Partial indexes (PostgreSQL)
```sql
CREATE INDEX idx_active_users ON users (email) WHERE deleted_at IS NULL;
-- Smaller index, faster queries for the common case
```

## Query Optimization

### N+1 Query Prevention
- **ORM**: Always use eager loading (`select_related`/`prefetch_related` in Django, `Preload` in GORM, `include` in Prisma)
- **GraphQL**: Use DataLoader pattern
- **Detection**: Log query count per request in development. Alert if > 20 queries per request.

### EXPLAIN ANALYZE (always)
Before deploying any query that touches > 10K rows:
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...
```
Look for: Seq Scan on large tables (need index), high actual vs estimated rows (stale statistics), nested loops with high row counts.

### Query anti-patterns
- `SELECT *` — select only needed columns
- `LIKE '%search%'` — use full-text search or trigram index
- `WHERE function(column) = value` — prevents index use; compute on the value side
- Subqueries that could be JOINs
- `OFFSET` for deep pagination — use cursor-based (`WHERE id > last_seen_id`)

## Migration Best Practices

### Safe migration order for zero-downtime deployments
1. **Add new column** (nullable, no default constraint)
2. **Deploy code** that writes to both old and new columns
3. **Backfill** existing rows
4. **Deploy code** that reads from new column
5. **Drop old column** (separate migration, separate deployment)

### Dangerous operations (require extra care)
- Adding NOT NULL constraint to existing column → add check constraint first, validate, then promote
- Renaming column → add new column, dual-write, migrate, drop old
- Adding index on large table → `CREATE INDEX CONCURRENTLY` (PostgreSQL)
- Changing column type → usually requires new column + migration

### Every migration must
- Be reversible (up + down)
- Be idempotent (safe to run twice)
- Be tested against production-like data volumes
- Have a rollback plan documented

## PostgreSQL-Specific Best Practices

- `VACUUM` and `ANALYZE` run regularly (autovacuum should be tuned, not disabled)
- Connection pooling via PgBouncer (not direct connections from app)
- Use `pg_stat_statements` to identify slow queries
- Partitioning for tables > 100M rows (range partition by date is most common)
- Advisory locks for application-level coordination
- `LISTEN/NOTIFY` for lightweight pub/sub before reaching for a message queue

## Redis Usage Patterns

- **Cache**: String keys with TTL. Cache-aside pattern. Prefix keys with service name.
- **Sessions**: Hash per session, set TTL to match session duration.
- **Rate limiting**: `INCR` + `EXPIRE` or Redis modules (redis-cell).
- **Queues**: Use Streams or BullMQ. Avoid LPUSH/RPOP for reliable queues.
- **Pub/Sub**: For real-time broadcasts. Messages are fire-and-forget (no persistence).
- **Distributed locks**: Redlock algorithm. Short TTL. Always release explicitly.

## Data Modeling Decisions

| Pattern | When to use |
|---------|------------|
| UUID primary keys | Distributed systems, public-facing IDs |
| Auto-increment IDs | Single database, internal only |
| Composite primary keys | Junction tables, natural keys |
| JSONB columns | Flexible attributes, user preferences, metadata |
| Separate tables | Structured data with relationships and constraints |
| Materialized views | Expensive aggregations queried frequently |