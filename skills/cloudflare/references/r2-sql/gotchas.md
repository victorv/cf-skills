# R2 SQL Gotchas

**Verified against the live engine, June 2026. The public docs lag the engine** — JOINs, subqueries, CTEs, set operations, and window functions all work now even where docs/limitations pages say "not supported." Re-verify with a live query if in doubt.

## What Now Works (don't trust stale "unsupported" claims)

JOINs (all types + multi-way), subqueries (IN/NOT IN/EXISTS/scalar/derived), multi-table CTEs (with JOINs), `SELECT DISTINCT`, `UNION`/`UNION ALL`/`INTERSECT`/`EXCEPT`, JSON functions, `EXPLAIN FORMAT JSON`, unpartitioned tables (OK for <1000 files), and the **full window-function set** (`ROW_NUMBER`/`RANK`/`DENSE_RANK`/`PERCENT_RANK`/`NTILE`/`CUME_DIST`, `LAG`/`LEAD` w/ offset+default, `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE`, aggregates over windows, `ROWS`/`RANGE`/`GROUPS` frames incl. `INTERVAL`, `QUALIFY`).

## What Does NOT Work

| Feature | Error / behavior | Workaround |
|---------|------------------|------------|
| `OFFSET` | `40003: OFFSET clause is not supported` | Cursor pagination (WHERE + ORDER BY) |
| Named `WINDOW w AS (...)` clause | `40003: WINDOW clause is not supported` | Inline `OVER (...)` (the only window feature missing) |
| `func(DISTINCT ...)` on aggregates | unsupported | `approx_distinct()` for distinct counts |
| `ARRAY_AGG` / `STRING_AGG` | blocked (memory safety) | none in R2 SQL |
| `LATERAL` derived tables | not supported | restructure subqueries |
| `UNNEST` / `PIVOT` / `UNPIVOT` | not supported | flatten at write time |
| `map_entries()` on stored columns | `80001` | `map_keys` / `map_values` / `map_extract` |
| INSERT / UPDATE / DELETE / DDL | `only read-only queries` | PySpark / PyIceberg / wrangler |
| `SELECT` without `FROM` | must reference a table | reference a table |

> No Workers binding — query the REST endpoint via `fetch()` ([patterns.md](patterns.md#dashboard-worker)), or use D1 / external DB for OLTP.

## Type Safety

```sql
-- ❌ wrong                          -- ✅ right
WHERE status = '200'                 WHERE status = 200
WHERE ts > '2026-01-01'              WHERE ts > '2026-01-01T00:00:00Z'   -- need time + tz
WHERE method = GET                   WHERE method = 'GET'
```

No implicit conversions. Timestamps must be RFC3339 with timezone; dates ISO 8601.

## Defaults & Behavior

- **LIMIT:** default 500, max 10,000 (use cursor pagination beyond that).
- **`now()` / `current_time()` quantized to 10 ms** (security measure).
- Wrangler needs `WRANGLER_R2_SQL_AUTH_TOKEN` — it does **not** reuse `wrangler login`.
- Open beta: R2 Storage **Admin Read & Write required even for read-only** queries.

## Performance

- **File count dominates latency** — enable automatic compaction.
- **Partition-filter + narrow time windows + always `LIMIT`.**
- **Multi-way JOINs on large tables** can exceed resource limits — filter heavily, join through dimension tables.
- Per-query `metrics` (`files_scanned`, `bytes_scanned`, `cache_hits`) are the primary observability signal — there is no dedicated R2 SQL GraphQL dataset. Full guidance: `https://developers.cloudflare.com/r2-sql/reference/limitations-best-practices/`.

## Debug Checklist

1. `wrangler r2 bucket catalog enable <bucket>` — catalog on?
2. `echo $WRANGLER_R2_SQL_AUTH_TOKEN` — token set?
3. `SHOW DATABASES` → `SHOW TABLES IN ns` → `DESCRIBE ns.table`
4. `SELECT COUNT(*) FROM ns.table` — data present?
5. Add filters incrementally; read `metrics` to tune.

## See Also

- [api.md](api.md) · [patterns.md](patterns.md) · [configuration.md](configuration.md)
