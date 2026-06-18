# Live eval kit — warehouse-analytics

You run this against a **real** `hermes-warehouse` MCP connection (the only way to validate true
end-to-end behavior and data correctness). It tells you whether the skill actually changes how Claude
behaves, and where it falls short, so we can iterate.

The prompts live in [`evals.json`](evals.json); this doc is how to run and grade them. Record what you
find in [`RESULTS_TEMPLATE.md`](RESULTS_TEMPLATE.md) (copy it per run) and bring it back.

---

## 1. Setup (once)

1. **Pick a test user + tenant** with a realistic access profile. For the prompts to exercise
   everything, the user ideally has:
   - at least **two schemas in different data sources** (for the cross-source case #2),
   - at least one **tenant-scoped** schema (the `:external_tenant_id` case #1),
   - and **no access** to at least one schema you can name (for the access-gap case #4).
2. **Connect the MCP** in a Claude Code session:
   ```bash
   claude mcp add --transport http hermes-warehouse "https://<tenant>.<domain>/api/mcp"
   ```
   The first tool call kicks off the Google OAuth flow in the browser. Sign in as the test user.
3. **Sanity check** the connection: ask Claude to "list the warehouse schemas I can access." You should
   see the test user's schemas. If it's empty, the user has no access profile — fix that first.

## 2. Map the prompts to your real data

The prompts use generic names ("live marketplace", `payments_live`). Before running, **substitute your
real schema/table names** so the questions are answerable:

- **#1** — point "new listings last month" at a real listings-style table (note whether its schema is
  tenant-scoped; that's what we want to test).
- **#2** — pick a "users" schema and a "payments/transactions" schema that genuinely live in **different
  data sources**.
- **#3** — pick schemas that hold signups, listings, and transactions.
- **#4** — replace `payments_live` with a schema the test user **genuinely cannot access**, so the gap
  is real (if they can access everything, this case can't fail).

Keep your substitutions in the results file so we can interpret the runs.

## 3. How to run each prompt

- **Fresh session per prompt** — so context from one doesn't leak into the next.
- **Same model** you intend to ship with.
- For each prompt, ideally run it **twice**:
  - **Baseline** — MCP connected, but the `warehouse-analytics` skill **not** installed/available. This
    isn't "no help" — Claude still has the tools' descriptions and the server's own short INSTRUCTIONS.
    It's the bar the skill has to beat.
  - **With skill** — the `hermes-warehouse` plugin installed so the skill is available.
  - If running both is too much friction, just do **with-skill** and grade against the rubric in
    absolute terms; the baseline is a nice-to-have that quantifies lift.
- Let Claude drive the tools itself; answer its clarifying questions as a normal user would (don't
  over-coach). Save the transcript.

## 4. Grading rubric

Score each criterion ✅ / ❌ from the transcript. A few are **auto-fail** signals (marked ⛔) — if you
see them, the skill needs work regardless of the rest. Tally met/total per prompt and jot a one-line
"what felt off."

### Prompt 1 — metric + trend (tenant scope)
- [ ] Called `list_schemas` (or `my_access`) **before** writing SQL — ⛔ if it guessed table names
- [ ] Confirmed the actual table + column names it used (drilled in)
- [ ] If the table's schema is tenant-scoped, the query includes the literal `:external_tenant_id` filter
- [ ] Used **aggregation** (COUNT / GROUP BY), not a `SELECT *` row dump
- [ ] Called `validate_query` before `run_query` on the non-trivial query
- [ ] Returned **both** months' figures + an explicit up/down (%) comparison
- [ ] Stated the time window and any assumptions ("last month" = which exact dates)
- [ ] ⛔ Did not attempt any write/DDL

### Prompt 2 — cross-source split
- [ ] Recognized users and payments are in **different data sources**
- [ ] ⛔ Did **not** present a single SQL joining across the two sources as if it would work
- [ ] If it tried one cross-source join and got rejected, it **recovered** (split into per-source queries)
- [ ] Ran one query per source and **combined the results itself**
- [ ] Defined "active" explicitly before computing
- [ ] Computed the average correctly (e.g. total revenue ÷ active-user count), aggregated not dumped
- [ ] Presented one clear number + caveats/assumptions

### Prompt 3 — dashboard
- [ ] Decomposed into discrete **widgets** (~3)
- [ ] Chose a sensible `widget_type` per widget (trend → line; "top cities" → bar, not pie)
- [ ] Queries use **rolling windows** (`DATEADD`/`CURRENT_DATE`/`DATE_TRUNC`), not hard-coded literal dates
- [ ] Previewed at least one widget's **actual numbers** via `run_query`
- [ ] Emitted widget specs in the **Report/Widget shape** (`title`, `widget_type`, `sql_query`, `config`)
- [ ] Each widget query obeys single-source + schema-qualified rules
- [ ] Was clear the MCP is **read-only** and saving the dashboard is a separate step (didn't imply it saved)

### Prompt 4 — access gap / my_access
- [ ] Called **`my_access`** (not only `list_schemas`)
- [ ] Reported the user's access **profiles** + accessible schemas
- [ ] Checked the inaccessible schema specifically (ideally `my_access check:"<schema>"`)
- [ ] Stated plainly it's an **access gap** and how to request it from an admin
- [ ] ⛔ Did **not** invent table names or try to query the inaccessible schema anyway

## 5. What to bring back

For each prompt, in `RESULTS_TEMPLATE.md`:
- the criteria that **failed** (and the ⛔ flags, if any),
- the **transcript** (or at least the tool-call sequence + final answer),
- a free-text **"what felt off"** — this is the most valuable input. Examples: "it asked three
  clarifying questions when one would do", "it thrashed on validate_query for 4 tries", "the dashboard
  config keys didn't match the query columns", "it was too verbose before showing the number."

I'll generalize the failures into skill changes (not one-off patches), we re-run, and repeat until it's
solid. Once behavior is good, we can also run the description-optimization loop to tune *when* the skill
fires.

## 6. Optional: capture transcripts on disk

If you want a tidy record, drop each run under:
```
evals/live-results/iteration-1/eval-<id>/{baseline,with_skill}/transcript.md
```
Otherwise pasting the relevant parts back to me works fine.
