# Query rules — the SQL contract for `run_query` / `validate_query`

This is the full contract the `hermes-warehouse` MCP server enforces. The point isn't to memorize
rules — it's to understand *why* each exists, so your first query usually passes and, when it
doesn't, you fix the right thing instead of thrashing.

Contents:
- [How the server runs your query](#how-the-server-runs-your-query)
- [The rules and why they exist](#the-rules-and-why-they-exist)
- [Rejection messages → how to fix them](#rejection-messages--how-to-fix-them)
- [Tenancy: scoping to one tenant](#tenancy-scoping-to-one-tenant)
- [One data source per query](#one-data-source-per-query)
- [The row cap and aggregation strategy](#the-row-cap-and-aggregation-strategy)
- [Worked examples](#worked-examples)

## How the server runs your query

Before executing, the server parses your SQL into an AST (it doesn't just string-match), checks every
rule below, **injects tenant scoping** for multi-tenant schemas, rewrites table names, and then
**wraps your statement** roughly like this:

```sql
SELECT * FROM ( <your query> ) AS mcp_query LIMIT 500
```

Two consequences worth holding onto:

- An inner `LIMIT` you write **does not** raise the ceiling — the outer `LIMIT 500` always wins. So
  `LIMIT` is for *shaping* (e.g. top-10), never for "getting everything".
- Your `ORDER BY` still matters: with the cap applied on top, ordering decides *which* rows survive
  truncation. If you want the top 10 by revenue, `ORDER BY revenue DESC` — don't rely on natural order.

`validate_query` runs every check **except** execution, so use it to iterate cheaply. Pass it the same
`tenant` you intend to run with, so tenant scoping is validated too.

## The rules and why they exist

| Rule | Why | What it means for you |
|------|-----|-----------------------|
| **Single statement** | One read, no script injection, predictable cost | Submit exactly one query — no `;`-separated statements |
| **`SELECT` only** | The warehouse is read-only by contract | No `INSERT/UPDATE/DELETE/MERGE`, no DDL — **not even inside a CTE** |
| **Schema-qualified tables** | Avoids ambiguity across many project schemas; names are validated against your access | Write `schema.table`, never a bare `table`. (CTE names you define are exempt.) |
| **Accessible tables only** | Server-side access-profile enforcement | You can only reference tables that appear in `list_schemas`. Others are rejected. |
| **One data source** | Tables in different connections can't be joined in one engine | Keep every table in a single schema/connection per query |
| **One tenant** | Multi-tenant schemas mix many tenants in the same tables; blending them is a data leak | Pass the `tenant` argument; the server injects the filter. One tenant per query (or `cross_tenant: true`) |
| **Row cap (500)** | Protects context and the warehouse | Aggregate and filter; treat row dumps as a smell |

CTEs (`WITH name AS (…)`) are fully supported and encouraged for readable multi-step logic. Only the
references to **real** tables need schema-qualifying; references to CTEs you defined in the same query
do not.

## Rejection messages → how to fix them

Rejections arrive as readable text (`invalid: …` from `validate_query`, `Query rejected: …` or
`Query failed: …` from `run_query`), never as crashes. Map the message to the fix:

| Message (substring) | Cause | Fix |
|---------------------|-------|-----|
| `only a single statement is allowed` | You sent more than one statement | Send one `SELECT`; drop extra statements and trailing `;`-separated parts |
| `only SELECT statements are allowed` | Top-level statement isn't a `SELECT` | Rewrite as a `SELECT` (wrap logic in a CTE if needed) |
| `data-modifying statements are not allowed …(found …)` | A write/DDL node appears (even in a CTE) | Remove the write; this surface is read-only |
| `table "X" must be schema-qualified — use schema.table` | A real table is referenced without its schema | Qualify it: `schema.X`. (If `X` was meant to be a CTE, define it in a `WITH`.) |
| `query must reference at least one accessible table (schema.table)` | No qualified, accessible table found | Reference at least one `schema.table` you can see in `list_schemas` |
| `you do not have access to schema.table` | Table exists but isn't in your access profiles | Use a table you can access, or ask an admin for that access profile — don't retry variants |
| `query references more than one data source; query one at a time` | Tables span two connections | Split into two queries (one per source) and join the results yourself |
| `this query reads tenant-partitioned data (…); specify exactly one tenant` | Multi-tenant schema, no tenant chosen | Pass `tenant: "<slug>"` (or a numeric id). To read all tenants, pass `cross_tenant: true` |
| `unknown tenant "X" in <schema>` | The tenant name didn't match the directory | Use one of the listed names, or query the schema's `tenants` table to find it; or pass the numeric id |
| `ambiguous tenant "X" in <schema>` | The name matched more than one tenant | Use the exact slug or the numeric id from the candidates listed |
| `this schema is tenant-scoped — filter rows with the :external_tenant_id placeholder` | A *schema-bound* tenant schema (rarer) needs the placeholder | Add `WHERE external_tenant_id = :external_tenant_id` (verbatim) |
| `query spans schemas bound to different external tenants; query one at a time` | Two schema-bound tenant schemas with different tenants | Query one such schema at a time |
| `warehouse connection pool exhausted — try again shortly` | Transient load | Wait briefly and retry the same query |

When the message names a column/table you thought existed, that's your cue to re-run `list_schemas`
(`table:"schema.table"`) and correct the name — don't keep guessing.

## Tenancy: scoping to one tenant

There are **two** kinds of tenant schema. Most of the time you mean the first.

### Row-level multi-tenant schemas (the common case)

One project's schema holds several tenants (brands / markets / portals) in the **same tables**,
distinguished by a tenant column. Aggregating without scoping blends them — a real data leak. So:

- **You don't write the filter.** Pass the **`tenant`** argument to `run_query`/`validate_query` — a
  tenant slug/name (e.g. `"north-region"`) or its numeric id — and the server injects
  `tenant = <that tenant>` into every reference to a tenant table, across CTEs, subqueries, and joins.
- If you touch tenant-partitioned data **without** `tenant`, the query is rejected
  (`specify exactly one tenant`). That's the signal to resolve the tenant, not to invent a filter.
- **Discover the tenants** by querying the schema's directory table (usually `schema.tenants`) — it
  isn't itself tenant-scoped, so it queries freely. Match the user's wording to a slug/name; if it's
  ambiguous, ask.
- **All tenants at once** is occasionally wanted ("by brand", "company-wide"). Only then pass
  **`cross_tenant: true`**, and make clear the figure spans every tenant.

```sql
-- run_query  sql: this,  tenant: "north-region"
SELECT status, COUNT(*) AS n
FROM marketplace.listings
WHERE created_at >= DATE '2026-05-01'
GROUP BY status;
-- the server runs it as: … WHERE created_at >= DATE '2026-05-01' AND listings.tenant_id = <id> …
```

Do **not** add `WHERE tenant_id = …` yourself — you don't need the id, and a hand-written filter is
redundant with (and can conflict with) the injected one.

### Schema-bound tenant schemas (rarer)

A few schemas are bound *wholly* to a single external tenant. For those, the server requires the
literal `:external_tenant_id` placeholder, which it substitutes with the bound value:

```sql
SELECT DATE(created_at) AS day, COUNT(*) AS n
FROM some_bound_schema.events
WHERE external_tenant_id = :external_tenant_id
  AND created_at >= DATE '2026-06-01'
GROUP BY DATE(created_at);
```

You won't guess wrong here: you only need the placeholder if a query is rejected asking for it. You
can't mix two schema-bound schemas with different tenants in one query.

## One data source per query

The warehouse is fronted by multiple connections (one per project's database). The engine can only
join tables that live in the *same* connection. If answering a question needs data from two projects:

1. Query source A, get its result.
2. Query source B, get its result.
3. Join/merge them yourself (in your reasoning or a follow-up computation) and present the combined
   answer. Aggregate within each query first — don't try to pull raw rows from both and join across
   the 500-row cap.

If you try to join across sources in one statement you'll get
`query references more than one data source; query one at a time`.

## The row cap and aggregation strategy

Every result is capped at 500 rows, applied *around* your whole query. This is a feature: it keeps you
honest about returning **answers**, not data dumps. Practical habits:

- **Aggregate.** `COUNT/SUM/AVG … GROUP BY` collapses millions of rows into the handful that answer the
  question.
- **Filter the window first.** Put the time/segment filter in `WHERE` so the engine isn't scanning all
  of history.
- **Rank when you want "top N".** `ORDER BY metric DESC LIMIT 10` — and remember the outer cap still
  applies, so keep N well under 500.
- **Watch the footer.** If `run_query` reports you hit the cap, the answer is **partial**. Don't present
  it as complete — tighten the aggregation or filter and rerun.

## Worked examples

Schema/table names below are **illustrative** — always discover the real ones with `list_schemas`.

**Example 1 — simple count, single schema**

Question: "How many listings were created last month?"

```sql
SELECT COUNT(*) AS listings_created
FROM marketplace.listings
WHERE created_at >= DATE '2026-05-01'
  AND created_at <  DATE '2026-06-01';
```

If `marketplace` is multi-tenant this is rejected until you pass a `tenant` — then it counts that one
tenant's listings.

**Example 2 — multi-tenant, grouped trend**

Question: "Daily new users last week, for the OLX Egypt brand."

You resolve the tenant first (ask, or check the directory), then pass it as `tenant` — **not** in SQL:

```sql
-- run_query  sql: this,  tenant: "olx-eg"
SELECT DATE(created_at) AS day, COUNT(*) AS new_users
FROM marketplace.users
WHERE created_at >= DATE '2026-06-09'
  AND created_at <  DATE '2026-06-16'
GROUP BY DATE(created_at)
ORDER BY day;
```

The server adds the single-tenant filter for you. Don't write `WHERE tenant_id = …`.

**Example 3 — top N with a CTE**

Question: "Top 10 cities by completed transactions this quarter." (single-tenant schema)

```sql
WITH completed AS (
  SELECT city_id
  FROM payments.transactions
  WHERE status = 'completed'
    AND created_at >= DATE '2026-04-01'
)
SELECT c.name AS city, COUNT(*) AS completed_txns
FROM completed
JOIN payments.cities c ON c.id = completed.city_id
GROUP BY c.name
ORDER BY completed_txns DESC
LIMIT 10;
```

Note: `completed` (the CTE) needs no schema; `payments.transactions` and `payments.cities`
(real tables) do — and both live in the same `payments` source.

**Example 4 — what gets rejected, and the fix**

```sql
-- ❌ rejected: bare table name
SELECT COUNT(*) FROM listings;
-- → table "listings" must be schema-qualified — use schema.table

-- ✅ fix
SELECT COUNT(*) FROM marketplace.listings;
```

```sql
-- ❌ rejected: multi-tenant schema, no tenant chosen
--   run_query  sql: SELECT COUNT(*) FROM marketplace.listings
-- → this query reads tenant-partitioned data (marketplace); specify exactly one tenant …

-- ✅ fix: pass the tenant argument (not a hand-written filter)
--   run_query  sql: SELECT COUNT(*) FROM marketplace.listings,  tenant: "olx-eg"
```

```sql
-- ❌ rejected: spans two data sources
SELECT u.id, t.amount
FROM marketplace.users u
JOIN payments.transactions t ON t.user_id = u.id;
-- → query references more than one data source; query one at a time

-- ✅ fix: two queries, then join the results yourself
-- (1) get user ids from marketplace, (2) get amounts from payments, (3) merge.
```

Always prefer `validate_query` to discover these before spending an execution.
