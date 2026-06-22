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
- [Step 6 — Downloadable report files](#step-6--downloadable-report-files-a-different-output-mode-than-the-in-chat-preview)
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
- **Scope to one tenant — outside the SQL.** A dashboard is for one tenant; don't bake a tenant filter
  into the widget query. Record the tenant on the report and pass it as the `tenant` argument when you
  preview/run each widget on a multi-tenant schema — the server injects the scoping. (The rarer
  schema-bound schemas still use the in-SQL `:external_tenant_id` placeholder; see
  [query-rules.md](query-rules.md).)
- **Aggregate to the grain.** A `line_chart` query returns one row per bucket; a `metric` query
  returns one row, one value. Don't return raw rows and chart them later — return the shaped result.
- **Order deterministically** (`ORDER BY day`, `ORDER BY metric DESC`) so the chart and the row-cap
  behave predictably.

The only runtime parameter is the **`tenant`** argument (it scopes multi-tenant schemas; the rarer
schema-bound schemas auto-substitute `:external_tenant_id`). There's no *general* parameter mechanism —
so a "parameterized" dashboard (e.g. a city filter) is expressed as a rolling expression in the SQL or
as separate widgets per segment, not as runtime parameters beyond `tenant`.

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
  "sql_query": "SELECT DATE_TRUNC('month', created_at) AS month, SUM(amount) AS revenue FROM payments.transactions WHERE created_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '11 months' GROUP BY 1 ORDER BY month",
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

- **Preview first — as numbers and visuals, not SQL.** Run each widget's query with `run_query` and
  show the user the actual figures so they confirm the metric is right *before* it's saved. If your
  environment can render charts, show the chart. Keep the SQL and the widget JSON out of the
  conversation — the user signs off on the numbers, not the query.
- **The widget spec is a machine handoff, not user-facing.** The JSON below is what gets sent to the
  report surface to build the dashboard; don't paste it at the user. Describe the dashboard in plain
  terms instead ("three tiles: active users, a weekly listings trend, and top cities by transactions").
- **Persisting is a separate step from querying.** The `hermes-warehouse` MCP is **read-only** — it has
  no tool to create reports or widgets. Saving a dashboard happens through Hermes' report/widget
  surface (the Report Manager: reports + widgets endpoints), not through this MCP. So you produce the
  validated widget specs as the handoff plus the previewed results; creating the persistent report is
  done via that surface. Be clear with the user about this boundary rather than implying the dashboard
  is saved when only the query has run.

## Step 6 — Downloadable report files (a different output mode than the in-chat preview)

"Preview and persist" above covers two destinations: a live chat preview, or a saved Hermes report.
There's a third thing users ask for that isn't either of those: **a file they can download, keep, or
send to someone** — signaled by words like "downloadable", "a file I can keep", "send me", "export
this", "save this as a report". Don't reuse the in-chat preview's markup for this — it will render
broken once it leaves the chat.

**Why it breaks:** the in-chat preview (built with the Visualizer/Imagine tool) is styled with CSS
variables the host page provides — `var(--color-background-secondary)`, `var(--color-text-primary)`,
etc. — specifically so it adapts to light/dark mode while running *inside* claude.ai. Extract that same
markup into a standalone `.html` file and open it outside the host: every `var(--color-...)` resolves to
nothing. Backgrounds, text colors, and borders all silently disappear — which looks like "the formatting
got stripped," but is really a missing CSS dependency.

**What to do instead:** when the deliverable is a downloadable file, build a second, separate,
self-contained HTML file rather than exporting the Visualizer widget:

- Write a real `<style>` block with literal values (hex colors, plain `px`/`rem` units) — no
  `var(--color-...)` references, since there's no host page to supply them.
- Keep the same content and chart logic; Chart.js still works the same way — load it from
  `cdnjs.cloudflare.com` via a `<script src="...">` tag exactly as in the preview.
- Save it with `create_file` to the outputs directory and hand it over with `present_files`, the same
  as any other generated file.

The in-chat preview and the downloadable file are two different artifacts answering two different
requests — build the preview to show the numbers *now*, and build the standalone file only when the
person actually asks to download, keep, or send something.

## Worked example

User: *"Can you build me a little dashboard tracking our marketplace — active users, new listings over
time, and where transactions are happening?"*

1. **Decompose** into three widgets: a headline active-users metric, a listings trend, a transactions-
   by-city comparison.
2. **Define metrics** (confirm with user): active = distinct users with a session in the last 30 days;
   listings trend = new listings per week, last 12 weeks; transactions by city = completed txns per
   city, last 30 days, top 10.
3. **Build + validate each query** (rolling windows, one source each), and for multi-tenant schemas pass
   the dashboard's tenant via the `tenant` argument — not in the SQL — then preview with `run_query`.
4. **Emit the report spec** (the behind-the-scenes handoff — not shown to the user):

```json
{
  "report": { "title": "Marketplace Health", "description": "Live marketplace KPIs", "tenant": "olx-eg" },
  "widgets": [
    {
      "title": "Active Users (last 30 days)",
      "widget_type": "metric",
      "sql_query": "SELECT COUNT(DISTINCT user_id) AS active_users FROM marketplace.sessions WHERE started_at >= DATEADD(day, -30, CURRENT_DATE)",
      "config": { "label": "Active Users (30d)" }
    },
    {
      "title": "New Listings per Week (12 weeks)",
      "widget_type": "line_chart",
      "sql_query": "SELECT DATE_TRUNC('week', created_at) AS week, COUNT(*) AS new_listings FROM marketplace.listings WHERE created_at >= DATEADD(week, -12, CURRENT_DATE) GROUP BY 1 ORDER BY week",
      "config": { "x_axis": "week", "y_axis": "new_listings", "label": "New Listings / Week" }
    },
    {
      "title": "Top Cities by Completed Transactions (30d)",
      "widget_type": "bar_chart",
      "sql_query": "SELECT c.name AS city, COUNT(*) AS completed_txns FROM payments.transactions t JOIN payments.cities c ON c.id = t.city_id WHERE t.status = 'completed' AND t.created_at >= DATEADD(day, -30, CURRENT_DATE) GROUP BY c.name ORDER BY completed_txns DESC LIMIT 10",
      "config": { "x_axis": "city", "y_axis": "completed_txns", "label": "Completed Transactions by City" }
    }
  ]
}
```

5. **Show the user** the previewed numbers and (if you can render them) the charts — described in plain
   terms, not as JSON. The spec above is the behind-the-scenes handoff. Explain that saving it as a
   live report is the next step via the report surface (this MCP only reads).

The report carries the **tenant** once (`"olx-eg"`); every multi-tenant widget is previewed/run with
that `tenant` argument, so the SQL stays tenant-free and the whole dashboard reads one tenant
consistently. (If a widget should span all tenants, mark it `cross_tenant` instead.)

Note how the third widget lives entirely in `payments` (one data source), while the first two live in
`marketplace` — that's fine, because **each widget is its own query**. The one-data-source rule applies
per query, not per dashboard.
