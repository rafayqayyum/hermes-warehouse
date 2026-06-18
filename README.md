# hermes-warehouse (Claude plugin)

A Claude plugin that lets people query the **Hermes data warehouse** (Amazon Redshift)
in plain language — safely. It bundles two halves that work together:

- **MCP server connection** (`hermes-warehouse`) — the authenticated, read-only SQL
  surface served by [`hermes-backend`](../hermes-backend) at `POST /api/mcp`. Every call
  is scoped to the signed-in user's **access profiles**, so a user can only ever see and
  query the schemas they're entitled to. The server validates SQL, enforces single
  schema-qualified `SELECT`s, injects tenant scoping, and caps result rows.
- **`warehouse-analytics` skill** — the playbook Claude follows to take a vague question
  ("how are listings doing this month?") through schema discovery, query construction,
  validation, execution, and presentation — ending in either a fetched answer or a
  dashboard spec.

The split is deliberate: **the MCP server is the security boundary** (it enforces access,
read-only, and validation server-side), and **the skill is the guidance layer** (it can be
wrong or jailbroken, so it never gates access — it only orchestrates within whatever the
server already allows).

## Install

This repo is **both the plugin and its own single-plugin marketplace** — it contains
`.claude-plugin/plugin.json` (the plugin) and `.claude-plugin/marketplace.json` (the catalog).
Once it's pushed to a git repo, anyone installs it with two commands:

```bash
# 1. Register the marketplace (GitHub shorthand, or a full git URL for self-hosted)
/plugin marketplace add <owner>/<repo>
#    e.g. /plugin marketplace add https://git.dubizzlelabs.run/data/hermes-warehouse.git

# 2. Install the plugin from it
/plugin install hermes-warehouse@dubizzle-plugins
```

The bundled `.mcp.json` is auto-discovered on install; on first use of a warehouse tool the
Google OAuth flow runs in the browser (one-time per user, token stored in the OS keychain).

### Local development / testing (no marketplace needed)

```bash
# Load straight from a local checkout:
claude --plugin-dir /path/to/hermes-warehouse

# Or exercise the real marketplace flow against the local repo before pushing:
/plugin marketplace add /path/to/hermes-warehouse
/plugin install hermes-warehouse@dubizzle-plugins
```

### Shipping updates

Bump `version` in `.claude-plugin/plugin.json`, commit, and push. Users pick it up via
`/plugin update` (or automatically at next startup). Omit `version` entirely to treat every
commit SHA as a new version (continuous updates).

## Configure the warehouse endpoint

The MCP endpoint is **per-tenant** — the tenant is conveyed by the subdomain. Set the URL
before first use (the `.mcp.json` reads it from an environment variable, falling back to a
placeholder you must replace):

```bash
export HERMES_WAREHOUSE_URL="https://<your-tenant>.<your-domain>/mcp"
```

The bundled `.mcp.json` already defaults to `https://hermes-be.dubizzlelabs.run/mcp`, so the env var is
only needed to point at a different tenant/host.

Or register it directly with the CLI:

```bash
claude mcp add --transport http hermes-warehouse "https://hermes-be.dubizzlelabs.run/mcp"
```

## Auth (handled automatically)

The endpoint uses **OAuth 2.1 + PKCE** with Google sign-in (Doorkeeper). On first use,
Claude self-registers (dynamic client registration) and runs the browser login flow — no
token pasting. After sign-in, the user only sees the schemas/tables their access profiles
grant.

## What you can ask

Anything that needs warehouse data: counts, trends, "top N", funnels, retention, KPIs,
"pull the data on X", "build me a dashboard for Y". The skill figures out which schema and
tables to use, writes a compliant query, validates and runs it, and explains the result.

## Layout

```
hermes-warehouse/
├── .claude-plugin/plugin.json          # plugin manifest
├── .mcp.json                           # hermes-warehouse MCP server (HTTP)
└── skills/
    └── warehouse-analytics/
        ├── SKILL.md                    # the end-to-end journey
        └── references/
            ├── query-rules.md          # the SQL contract + worked examples
            └── dashboards.md           # question → reusable query → dashboard
```
