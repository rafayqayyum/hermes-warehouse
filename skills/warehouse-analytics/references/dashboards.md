# From a question to a dashboard

When a user wants something **durable** — "a dashboard", "a report", "track this every week",
"add this as a chart" — you're no longer answering a one-off question. You're defining a reusable
artifact. This reference covers how to do that cleanly and produce output that maps directly onto
Hermes' report/widget model.

Contents:
- [The model: Reports and Widgets](#the-model-reports-and-widgets)
- [Step 1 — Nail the metric](#step-1--nail-the-metric)
- [Step 2 — Make the query durable](#step-2--make-the-query-durable)
- [Step 3 — Pick the widget type](#step-3--pick-the-widget-type)
- [Step 4 — Produce the widget spec](#step-4--produce-the-widget-spec)
- [Step 5 — Preview and persist](#step-5--preview-and-persist)
- [Worked example](#worked-example)

## The model: Reports and Widgets

A dashboard in Hermes is a **Report** that contains one or more **Widgets**. Each widget is a single
visualization backed by one query:

- `title` — what the widget shows
- `widget_type` — one of `metric`, `table`, `line_chart`, `bar_chart`, `pie_chart`
- `sql_query` — the finalized SELECT (must obey every rule in [query-rules.md](query-rules.md))
- `config` — visualization settings, typically `{ x_axis, y_axis, label }` (plus any series/format hints)
- `position` — ordering within the report

So "build me a dashboard for X" decomposes into: define the report, then define each widget as
*(metric + query + chart type + config)*. Build widgets one at a time, validating each query as you go.

## Step 1 — Nail the metric

A vague metric makes a useless dashboard. Before writing a widget's SQL, state precisely:

- **Definition** — exactly what's measured. "Active users" → e.g. *distinct users with ≥1 session in
  the period*. Write the definition down so the widget title and the SQL agree.
- **Grain** — one number (`metric`), one row per time bucket (`line_chart`), or one row per category
  (`bar_chart`/`pie_chart`/`table`)?
- **Window** — and crucially, **rolling vs. fixed**. A dashboard is re-run over time, so prefer a
  rolling window ("last 30 days", "last 12 complete months") over a hard-coded date, unless the user
  explicitly wants a fixed historical snapshot.
- **Filters/segments** — baked into the widget, or offered as a breakdown across widgets.

If any of these is ambiguous and would change the result, confirm it with the user now — it's much
cheaper than rebuilding the dashboard later.

## Step 2 — Make the query durable

A one-off query can hard-code `DATE '2026-05-01'`; a dashboard query should age well:

- **Use rolling date expressions** so the widget stays correct when re-run. On Redshift, e.g.
  `WHERE created_at >= DATEADD(day, -30, CURRENT_DATE)` or
  `created_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '11 months'` for a trailing-12-months
  view. Validate the exact expression with `validate_query` — Redshift's date functions differ from
  other engines.
- **Keep tenant scoping.** If the schema is tenant-scoped, the widget query still needs
  `WHERE external_tenant_id = :external_tenant_id`; it carries over unchanged.
- **Aggregate to the grain.** A `line_chart` query returns one row per bucket; a `metric` query
  returns one row, one value. Don't return raw rows and chart them later — return the shaped result.
- **Order deterministically** (`ORDER BY day`, `ORDER BY metric DESC`) so the chart and the row-cap
  behave predictably.

There is **no general query parameter mechanism** on this server beyond the auto-substituted
`:external_tenant_id`. So a "parameterized" dashboard (e.g. a city filter) is expressed as either a
rolling expression in the SQL, or as separate widgets per segment — not as runtime parameters you pass
to `run_query`.

## Step 3 — Pick the widget type

Match the chart to the question's shape — this is the single biggest driver of whether a dashboard
reads well:

| Question shape | widget_type | Query returns |
|----------------|-------------|---------------|
| A single headline number ("how many X right now") | `metric` | one row, one value |
| A trend over time ("X per day/week/month") | `line_chart` | one row per time bucket, ordered by time |
| Comparing categories ("X by city/category/status") | `bar_chart` | one row per category, usually ordered by the value |
| Parts of a whole, few slices ("share of X by type") | `pie_chart` | one row per slice — use sparingly, only for ≤~6 slices |
| Detailed rows the user needs to scan | `table` | the rows themselves (mind the 500 cap) |

When unsure between `bar_chart` and `pie_chart`, prefer `bar_chart` — it stays readable as categories
grow and supports more honest comparison.

## Step 4 — Produce the widget spec

Emit each widget in the shape the report/widget model expects, so it can be created directly. Use this
structure:

```json
{
  "title": "Monthly Revenue (last 12 months)",
  "widget_type": "line_chart",
  "sql_query": "SELECT DATE_TRUNC('month', created_at) AS month, SUM(amount) AS revenue FROM payments_live.transactions WHERE created_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '11 months' GROUP BY 1 ORDER BY month",
  "config": { "x_axis": "month", "y_axis": "revenue", "label": "Monthly Revenue" }
}
```

For a `metric` widget, `config` is minimal (`{ "label": "Active Users (30d)" }`) and the query returns
a single value. For `table`, `config` lists the columns to show. Keep `config` keys aligned with the
column names the query actually returns — the chart binds to those names.

A full dashboard is a small report object with an ordered list of these widgets:

```json
{
  "report": { "title": "Marketplace Health", "description": "Weekly KPIs for the live marketplace" },
  "widgets": [ /* widget objects, in display order */ ]
}
```

## Step 5 — Preview and persist

- **Preview first.** Run each widget's query with `run_query` and show the user the actual numbers, so
  they confirm the metric is right *before* it becomes a saved widget. If your environment can render
  charts, show a quick visual of the shaped result so they see what the widget will look like.
- **Persisting is a separate step from querying.** The `hermes-warehouse` MCP is **read-only** — it has
  no tool to create reports or widgets. Saving a dashboard happens through Hermes' report/widget
  surface (the Report Manager: reports + widgets endpoints), not through this MCP. So your deliverable
  here is the **validated widget specs above** plus the previewed results; creating the persistent
  report is done via that surface (or handed to the user/FE). Be clear with the user about this
  boundary rather than implying the dashboard is saved when only the query has run.

## Worked example

User: *"Can you build me a little dashboard tracking our marketplace — active users, new listings over
time, and where transactions are happening?"*

1. **Decompose** into three widgets: a headline active-users metric, a listings trend, a transactions-
   by-city comparison.
2. **Define metrics** (confirm with user): active = distinct users with a session in the last 30 days;
   listings trend = new listings per week, last 12 weeks; transactions by city = completed txns per
   city, last 30 days, top 10.
3. **Build + validate each query** (rolling windows, tenant filter if needed, one source each), preview
   with `run_query`.
4. **Emit the report spec:**

```json
{
  "report": { "title": "Marketplace Health", "description": "Live marketplace KPIs" },
  "widgets": [
    {
      "title": "Active Users (last 30 days)",
      "widget_type": "metric",
      "sql_query": "SELECT COUNT(DISTINCT user_id) AS active_users FROM phoenix_backend_live.sessions WHERE external_tenant_id = :external_tenant_id AND started_at >= DATEADD(day, -30, CURRENT_DATE)",
      "config": { "label": "Active Users (30d)" }
    },
    {
      "title": "New Listings per Week (12 weeks)",
      "widget_type": "line_chart",
      "sql_query": "SELECT DATE_TRUNC('week', created_at) AS week, COUNT(*) AS new_listings FROM phoenix_backend_live.listings WHERE external_tenant_id = :external_tenant_id AND created_at >= DATEADD(week, -12, CURRENT_DATE) GROUP BY 1 ORDER BY week",
      "config": { "x_axis": "week", "y_axis": "new_listings", "label": "New Listings / Week" }
    },
    {
      "title": "Top Cities by Completed Transactions (30d)",
      "widget_type": "bar_chart",
      "sql_query": "SELECT c.name AS city, COUNT(*) AS completed_txns FROM payments_live.transactions t JOIN payments_live.cities c ON c.id = t.city_id WHERE t.status = 'completed' AND t.created_at >= DATEADD(day, -30, CURRENT_DATE) GROUP BY c.name ORDER BY completed_txns DESC LIMIT 10",
      "config": { "x_axis": "city", "y_axis": "completed_txns", "label": "Completed Transactions by City" }
    }
  ]
}
```

5. **Show the user** the previewed numbers + the spec, and explain that saving it as a live report is
   the next step via the report surface (this MCP only reads).

Note how the third widget lives entirely in `payments_live` (one data source), while the first two live
in `phoenix_backend_live` — that's fine, because **each widget is its own query**. The one-data-source
rule applies per query, not per dashboard.
