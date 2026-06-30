# hermes-insights (Claude plugin)

A Claude plugin that lets people query the **Hermes data warehouse** (Amazon Redshift)
in plain language — safely. It bundles two halves that work together:

- **MCP server connection** (`hermes-warehouse`) — the authenticated, read-only SQL
  surface served by [`hermes-backend`](../hermes-backend) at `POST /mcp`. Every call
  is scoped to the signed-in user's **access profiles**, so a user can only ever see and
  query the schemas they're entitled to. The server validates SQL, enforces single
  schema-qualified `SELECT`s, injects tenant scoping, and caps result rows. It also exposes
  **knowledge-base search** (`search_knowledge_base`) — semantic search over the curated
  metric definitions attached to each schema, gated by the same access profiles, so one
  connector surfaces every knowledge base the user can reach (no per-corpus connector needed).
- **`hermes-analyst` skill** — the playbook Claude follows to take a vague question
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
/plugin install hermes-insights@hermes-marketplace
```

The bundled `.mcp.json` is auto-discovered on install; on first use of a warehouse tool the
Google OAuth flow runs in the browser (one-time per user, token stored in the OS keychain).

### Local development / testing (no marketplace needed)

```bash
# Load straight from a local checkout:
claude --plugin-dir /path/to/hermes-warehouse

# Or exercise the real marketplace flow against the local repo before pushing:
/plugin marketplace add /path/to/hermes-warehouse
/plugin install hermes-insights@hermes-marketplace
```

### Shipping updates

Bump `version` in `.claude-plugin/plugin.json`, commit, and push. Users pick it up via
`/plugin update` (or automatically at next startup). Omit `version` entirely to treat every
commit SHA as a new version (continuous updates).

## Configure the warehouse endpoint

The bundled `.mcp.json` points at `https://hermes-be.dubizzlelabs.run/mcp` and pins the OAuth
**client ID** (a public PKCE client — not a secret). To point at a different host/tenant, edit the
`url` in `.mcp.json`, or register via the CLI:

```bash
claude mcp add --transport http hermes-warehouse "https://<your-tenant>.<your-domain>/mcp"
```

### Adding it as a custom connector (Claude.ai / Desktop)

If you add the server directly instead of via the plugin, enter **literal** values — the connector
dialog does not expand `${VAR}` templates:

| Field | Value |
|---|---|
| URL | `https://hermes-be.dubizzlelabs.run/mcp` |
| OAuth Client ID | `Yo_PaFk7UDl9g5Zwjh5qLDqnsLKSOAJ9NIOxVDl0gQg` |
| OAuth Client Secret | *(leave blank — public PKCE client)* |

## Auth

The endpoint uses **OAuth 2.1 + PKCE** with Google sign-in (Doorkeeper). The server requires a
**pre-registered** OAuth client (dynamic client registration is **not** enabled), which is why the
client ID is bundled in `.mcp.json` and entered in the connector dialog — there is no client secret
(public client). The browser login runs once; after sign-in, the user only sees the schemas/tables
their access profiles grant.

## What you can ask

Anything that needs warehouse data: counts, trends, "top N", funnels, retention, KPIs,
"pull the data on X", "build me a dashboard for Y". The skill figures out which schema and
tables to use, looks up what each metric *means* in that schema's knowledge base, writes a
compliant query, validates and runs it, and explains the result.

## Layout

```
hermes-warehouse/
├── .claude-plugin/plugin.json          # plugin manifest
├── .mcp.json                           # hermes-warehouse MCP server (HTTP)
└── skills/
    └── hermes-analyst/
        ├── SKILL.md                    # the end-to-end journey
        └── references/
            ├── query-rules.md          # the SQL contract + worked examples
            └── dashboards.md           # question → reusable query → dashboard
```
