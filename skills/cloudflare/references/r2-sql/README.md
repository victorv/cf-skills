# Cloudflare R2 SQL

Serverless, distributed, **read-only** query engine (Apache DataFusion) for Apache Iceberg tables in R2 Data Catalog.

## Documentation

For full function lists, data types, and pricing, **retrieve the live docs** — use the `cloudflare-docs` MCP/search tool if available, otherwise `webfetch`.

| Topic | URL |
|-------|-----|
| Overview / get started | `https://developers.cloudflare.com/r2-sql/get-started/` |
| Query data | `https://developers.cloudflare.com/r2-sql/query-data/` |
| SQL reference | `https://developers.cloudflare.com/r2-sql/sql-reference/` |
| Aggregate functions | `https://developers.cloudflare.com/r2-sql/sql-reference/aggregate-functions/` |
| Scalar functions | `https://developers.cloudflare.com/r2-sql/sql-reference/scalar-functions/` |
| Complex types | `https://developers.cloudflare.com/r2-sql/sql-reference/complex-types/` |
| Limitations & best practices | `https://developers.cloudflare.com/r2-sql/reference/limitations-best-practices/` |
| Wrangler commands | `https://developers.cloudflare.com/r2-sql/reference/wrangler-commands/` |
| Pricing | `https://developers.cloudflare.com/r2-sql/platform/pricing/` |

## Connection Values

| Value | Format |
|-------|--------|
| REST endpoint | `https://api.sql.cloudflarestorage.com/api/v1/accounts/{ACCOUNT_ID}/r2-sql/query/{BUCKET}` |
| Wrangler | `npx wrangler r2 sql query "{WAREHOUSE}" "<SQL>"` with `WRANGLER_R2_SQL_AUTH_TOKEN` set |
| Warehouse | `{ACCOUNT_ID}_{BUCKET}` |

> The REST endpoint is `api.sql.cloudflarestorage.com` — **not** `api.cloudflare.com/.../r2/sql`.

## Quick Start

```bash
npx wrangler r2 bucket catalog enable my-bucket           # 1. enable catalog
export WRANGLER_R2_SQL_AUTH_TOKEN=<r2-token>              # 2. auth (Admin R&W + R2 SQL Read)
npx wrangler r2 sql query "$ACCOUNT_ID"_my-bucket \
  "SELECT * FROM default.my_table LIMIT 10"                # 3. query
```

## Quick Reference of What's Supported 

✅ `SELECT [DISTINCT]`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`
✅ **JOINs** (INNER/LEFT/RIGHT/FULL OUTER/CROSS/implicit, multi-way)
✅ **Subqueries** (IN, EXISTS, scalar, derived) · **CTEs** (multi-table, with JOINs)
✅ **Set ops** (UNION/UNION ALL, INTERSECT, EXCEPT)
✅ **Window functions** — full set + `QUALIFY` (inline `OVER (...)` only)
✅ Aggregate + scalar + JSON functions, complex types (struct/array/map), `EXPLAIN [FORMAT JSON]`

❌ `OFFSET`, named `WINDOW` clause, `func(DISTINCT ...)` on aggregates, `ARRAY_AGG`/`STRING_AGG`, `LATERAL`, `UNNEST`/`PIVOT`, INSERT/UPDATE/DELETE/DDL, `SELECT` without `FROM`

Detail + workarounds: [api.md](api.md), [gotchas.md](gotchas.md).

## When to Use

**Use for:** SQL analytics over Iceberg (logs, BI, fraud, ad-hoc), multi-cloud queries without egress, dashboards (query from a Worker via HTTP).

**Don't use for:** writes (use PySpark/PyIceberg), real-time OLTP (<100 ms), or the few unsupported features above (use PySpark).

## No Workers Binding

There is no `env.R2_SQL` binding. Query from a Worker via `fetch()` to the REST endpoint with the token as a secret (see [patterns.md](patterns.md#dashboard-worker)).

## Reading Order

1. [configuration.md](configuration.md) — enable catalog, tokens, env setup
2. [api.md](api.md) — SQL syntax templates, verified JOIN/window examples, response format, data types
3. [patterns.md](patterns.md) — CLI/REST/Worker queries, use cases, pagination, performance
4. [gotchas.md](gotchas.md) — what works vs. not, performance, troubleshooting

## See Also

- [r2-data-catalog](../r2-data-catalog/) — PyIceberg/PySpark, table management
- [pipelines](../pipelines/) — streaming ingest into queryable tables
