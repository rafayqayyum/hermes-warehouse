# Query rules — the SQL contract for `run_query` / `validate_query`

This is the full contract the `hermes-warehouse` MCP server enforces. The point isn't to memorize
rules — it's to understand *why* each exists, so your first query usually passes and, when it
doesn't, you fix the right thing instead of thrashing.

Contents:
- [How the server runs your query](#how-the-server-runs-your-query)
- [The rules and why they exist](#the-rules-and-why-they-exist)
- [Rejection messages → how to fix them](#rejection-messages--how-to-fix-them)
- [Tenant-scoped schemas](#tenant-scoped-schemas)
- [One data source per query](#one-data-source-per-query)
- [The row cap and aggregation strategy](#the-row-cap-and-aggregation-strategy)
- [Worked examples](#worked-examples)

## How the server runs your query

Before executing, the server parses your SQL into an AST (it doesn't just string-match), checks every
rule below, rewrites tenant placeholders, and then **wraps your statement** roughly like this:

```sql
SELECT * FROM ( <your query> ) AS mcp_query LIMIT 500
```

Two consequences worth holding onto:

- An inner `LIMIT` you write **does not** raise the ceiling — the outer `LIMIT 500` always wins. So
  `LIMIT` is for *shaping* (e.g. top-10), never for "getting everything".
- Your `ORDER BY` still matters: with the cap applied on top, ordering decides *which* rows survive
  truncation. If you want the top 10 by revenue, `ORDER BY revenue DESC` — don't rely on natural order.

`validate_query` runs every check **except** execution, so use it to iterate cheaply.

## The rules and why they exist

| Rule | Why | What it means for you |
|------|-----|-----------------------|
| **Single statement** | One read, no script injection, predictable cost | Submit exactly one query — no `;`-separated statements |
| **`SELECT` only** | The warehouse is read-only by contract | No `INSERT/UPDATE/DELETE/MERGE`, no DDL — **not even inside a CTE** |
| **Schema-qualified tables** | Avoids ambiguity across many project schemas; names are validated against your access | Write `schema.table`, never a bare `table`. (CTE names you define are exempt.) |
| **Accessible tables only** | Server-side access-profile enforcement | You can only reference tables that appear in `list_schemas`. Others are rejected. |
| **One data source** | Tables in different connections can't be joined in one engine | Keep every table in a single schema/connection per query |
| **Tenant scoping** | Multi-tenant schemas must not leak across tenants | Add the `:external_tenant_id` filter when told to |
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
| `this schema is tenant-scoped — filter rows with the :external_tenant_id placeholder` | Missing tenant filter | Add `WHERE external_tenant_id = :external_tenant_id` (verbatim placeholder) |
| `query spans schemas bound to different external tenants; query one at a time` | Two tenant-scoped schemas with different tenants | Query one tenant-scoped schema at a time |
| `warehouse connection pool exhausted — try again shortly` | Transient load | Wait briefly and retry the same query |

When the message names a column/table you thought existed, that's your cue to re-run `list_schemas`
(`table:"schema.table"`) and correct the name — don't keep guessing.

## Tenant-scoped schemas

Some schemas hold rows for multiple external tenants in the same tables. For those, the server
**requires** you to scope rows to the caller's tenant, and it fills in the value for you. You write the
placeholder literally:

```sql
SELECT status, COUNT(*) AS n
FROM phoenix_backend_live.listings
WHERE external_tenant_id = :external_tenant_id
  AND created_at >= DATE '2026-05-01'
GROUP BY status;
```

You do **not** know or hard-code the tenant value — `:external_tenant_id` is replaced server-side with
the authenticated tenant. If you forget it on a tenant-scoped schema, you'll get the
`this schema is tenant-scoped …` rejection; just add the filter and re-validate. You can't mix two
tenant-scoped schemas bound to different tenants in one query.

## One data source per query

The warehouse is fronted by multiple connections (one per project's database). The engine can only
join tables that live in the *same* connection. If answering a question needs data from two projects:

1. Query source A, get its result.
2. Query source B, get its result.
3. Join/merge them yourself (in your reasoning or a follow-up computation) and present the combined
   answer.

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

**Example 1 — simple count, single schema**

Question: "How many listings were created last month?"

```sql
SELECT COUNT(*) AS listings_created
FROM phoenix_backend_live.listings
WHERE created_at >= DATE '2026-05-01'
  AND created_at <  DATE '2026-06-01';
```

**Example 2 — tenant-scoped, grouped trend**

Question: "Daily new users last week, for our tenant."

```sql
SELECT DATE(created_at) AS day, COUNT(*) AS new_users
FROM phoenix_backend_live.users
WHERE external_tenant_id = :external_tenant_id
  AND created_at >= DATE '2026-06-09'
  AND created_at <  DATE '2026-06-16'
GROUP BY DATE(created_at)
ORDER BY day;
```

**Example 3 — top N with a CTE**

Question: "Top 10 cities by completed transactions this quarter."

```sql
WITH completed AS (
  SELECT city_id
  FROM payments_live.transactions
  WHERE status = 'completed'
    AND created_at >= DATE '2026-04-01'
)
SELECT c.name AS city, COUNT(*) AS completed_txns
FROM completed
JOIN payments_live.cities c ON c.id = completed.city_id
GROUP BY c.name
ORDER BY completed_txns DESC
LIMIT 10;
```

Note: `completed` (the CTE) needs no schema; `payments_live.transactions` and `payments_live.cities`
(real tables) do — and both live in the same `payments_live` source.

**Example 4 — what gets rejected, and the fix**

```sql
-- ❌ rejected: bare table name
SELECT COUNT(*) FROM listings;
-- → table "listings" must be schema-qualified — use schema.table

-- ✅ fix
SELECT COUNT(*) FROM phoenix_backend_live.listings;
```

```sql
-- ❌ rejected: spans two data sources
SELECT u.id, t.amount
FROM phoenix_backend_live.users u
JOIN payments_live.transactions t ON t.user_id = u.id;
-- → query references more than one data source; query one at a time

-- ✅ fix: two queries, then join the results yourself
-- (1) get user ids from phoenix_backend_live, (2) get amounts from payments_live, (3) merge.
```

Always prefer `validate_query` to discover these before spending an execution.
