<p align="center">
  <img src="assets/logo-600.png" alt="Snowflake MCP" width="600">
</p>

# Snowflake MCP Server

A [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server that enables AI agents to execute SQL queries against Snowflake databases.

> **Fork note.** This is a fork of [`faressoft/snowflake-mcp`](https://github.com/faressoft/snowflake-mcp),
> which supports SSO (browser) and username/password auth only. This fork adds:
>
> - **Programmatic Access Token (PAT)** auth — `SNOWFLAKE_AUTHENTICATOR=programmatic_access_token`
> - **Key-pair / JWT** auth — `SNOWFLAKE_AUTHENTICATOR=snowflake_jwt`
> - **OAuth access token** auth — `SNOWFLAKE_AUTHENTICATOR=oauth`
> - **Secure defaults**: readonly mode ON unless explicitly disabled, hardened
>   write-statement blocking, identifier validation on all table/schema inputs,
>   SDK logging off (no stray `snowflake.log` in your working directory), and
>   automatic reconnect when a cached session goes stale
>
> Install from this repo, not from npm: the published `snowflake-mcp` package on npm is
> upstream's and does **not** include the auth methods above. This fork is versioned `1.2.0+`
> to keep the two distinguishable.

Users can use natural language to query Snowflake databases, like:

- "Get me the top 10 products by revenue"
- "Show me the total revenue for the last 30 days"
- "Describe the structure of the orders table"
- "Explore the database and summarize what data is available"
- "Build a query to find customers who haven't ordered in 90 days"

## Features

- Execute SQL queries directly from AI agents like Cursor, Claude Desktop, etc.
- **Schema Discovery**: Browse databases, schemas, tables, and views
- **Table Inspection**: Describe table structures, view sample data, check row counts
- **Query Safety**: Readonly mode, row limits, and query timeouts
- **Multiple Output Formats**: Table, JSON, or CSV
- **Query Explanation**: Get execution plans for queries
- **MCP Prompts**: Guided workflows for common tasks
- Multiple auth methods: SSO (browser), username/password, programmatic access token (PAT), key-pair/JWT, OAuth token
- Configurable default warehouse, database, schema, and role

## Prerequisites

- Node.js 18 or later

## Installation

There is no build step — the compiled server is committed to `dist/`, so it runs straight
from a clone, an `npx` fetch, or a tarball install.

**Recommended: no install at all.** Point your MCP client at the fork via `npx`
(all the config examples below do this):

```json
"command": "npx",
"args": ["-y", "github:mwilston/snowflake-mcp"]
```

To pick up fork updates later, clear the npx cache: `rm -rf ~/.npm/_npx`.

**If you want a global `snowflake-mcp` binary**, install from the tarball URL:

```bash
npm install -g https://github.com/mwilston/snowflake-mcp/archive/refs/heads/master.tar.gz
```

> **Do not use `npm install -g github:mwilston/snowflake-mcp`.** npm 10 has a bug
> where a *global* install from a git spec creates a symlink into npm's cache temp
> directory instead of a real install, leaving a broken or dangling binary. The
> tarball URL above installs the identical code correctly. (Non-global installs
> and `npx` are unaffected.)

To work on the server locally:

```bash
git clone https://github.com/mwilston/snowflake-mcp.git && cd snowflake-mcp && npm install && npm run build && npm link
```

## Usage

### Cursor

Add to your Cursor MCP settings (`~/.cursor/mcp.json`):

#### SSO Authentication (Recommended)

```json
{
  "mcpServers": {
    "snowflake": {
      "command": "npx",
      "args": ["-y", "github:mwilston/snowflake-mcp"],
      "env": {
        "SNOWFLAKE_ACCOUNT": "your-org-your-account",
        "SNOWFLAKE_USERNAME": "your-username",
        "SNOWFLAKE_AUTHENTICATOR": "externalbrowser",
        "SNOWFLAKE_WAREHOUSE": "your-warehouse",
        "SNOWFLAKE_DATABASE": "your-database",
        "SNOWFLAKE_SCHEMA": "your-schema"
      }
    }
  }
}
```

A browser window will open for authentication on first query.

#### Password Authentication

```json
{
  "mcpServers": {
    "snowflake": {
      "command": "npx",
      "args": ["-y", "github:mwilston/snowflake-mcp"],
      "env": {
        "SNOWFLAKE_ACCOUNT": "your-org-your-account",
        "SNOWFLAKE_USERNAME": "your-username",
        "SNOWFLAKE_PASSWORD": "your-password",
        "SNOWFLAKE_ROLE": "your-role",
        "SNOWFLAKE_WAREHOUSE": "your-warehouse",
        "SNOWFLAKE_DATABASE": "your-database",
        "SNOWFLAKE_SCHEMA": "your-schema",
        "SNOWFLAKE_READONLY": "true"
      }
    }
  }
}
```

#### Programmatic Access Token (PAT) Authentication

Use this when SSO can't be used (headless environments, CI, shared tooling). Generate the
token in Snowsight under your user's **Programmatic access tokens**. Requires
`snowflake-sdk >= 2.4.0`, which handles the `PROGRAMMATIC_ACCESS_TOKEN` authenticator natively.

```json
{
  "mcpServers": {
    "snowflake": {
      "command": "npx",
      "args": ["-y", "github:mwilston/snowflake-mcp"],
      "env": {
        "SNOWFLAKE_ACCOUNT": "your-org-your-account",
        "SNOWFLAKE_USERNAME": "your-username",
        "SNOWFLAKE_AUTHENTICATOR": "programmatic_access_token",
        "SNOWFLAKE_TOKEN": "your-pat",
        "SNOWFLAKE_ROLE": "your-role",
        "SNOWFLAKE_WAREHOUSE": "your-warehouse",
        "SNOWFLAKE_READONLY": "true"
      }
    }
  }
}
```

#### Key-Pair (JWT) Authentication

Set either `SNOWFLAKE_PRIVATE_KEY_PATH` (path to a PEM/PKCS8 file) or `SNOWFLAKE_PRIVATE_KEY`
(the inline key body). `SNOWFLAKE_PRIVATE_KEY_PASS` is only needed for encrypted keys.

```json
{
  "mcpServers": {
    "snowflake": {
      "command": "npx",
      "args": ["-y", "github:mwilston/snowflake-mcp"],
      "env": {
        "SNOWFLAKE_ACCOUNT": "your-org-your-account",
        "SNOWFLAKE_USERNAME": "your-username",
        "SNOWFLAKE_AUTHENTICATOR": "snowflake_jwt",
        "SNOWFLAKE_PRIVATE_KEY_PATH": "/absolute/path/to/rsa_key.p8",
        "SNOWFLAKE_WAREHOUSE": "your-warehouse",
        "SNOWFLAKE_READONLY": "true"
      }
    }
  }
}
```

#### OAuth Token Authentication

For tokens issued by a Snowflake OAuth security integration. For PATs prefer
`programmatic_access_token` above.

```json
{
  "mcpServers": {
    "snowflake": {
      "command": "npx",
      "args": ["-y", "github:mwilston/snowflake-mcp"],
      "env": {
        "SNOWFLAKE_ACCOUNT": "your-org-your-account",
        "SNOWFLAKE_USERNAME": "your-username",
        "SNOWFLAKE_AUTHENTICATOR": "oauth",
        "SNOWFLAKE_TOKEN": "your-oauth-access-token"
      }
    }
  }
}
```

### Claude Desktop

Add to your Claude Desktop config (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "snowflake": {
      "command": "npx",
      "args": ["-y", "github:mwilston/snowflake-mcp"],
      "env": {
        "SNOWFLAKE_ACCOUNT": "your-org-your-account",
        "SNOWFLAKE_USERNAME": "your-username",
        "SNOWFLAKE_AUTHENTICATOR": "externalbrowser"
      }
    }
  }
}
```

## Configuration

| Variable                  | Required    | Default           | Description                                                                |
| ------------------------- | ----------- | ----------------- | -------------------------------------------------------------------------- |
| `SNOWFLAKE_ACCOUNT`       | Yes         | -                 | Your Snowflake account identifier (e.g., `ORG-ACCOUNT`)                    |
| `SNOWFLAKE_USERNAME`      | Yes         | -                 | Snowflake username                                                         |
| `SNOWFLAKE_AUTHENTICATOR` | No          | inferred          | `externalbrowser` (SSO), `snowflake` (password), `programmatic_access_token` (PAT), `snowflake_jwt` (key-pair), `oauth`. When unset: PAT if `SNOWFLAKE_TOKEN` is set, key-pair if a private key is set, otherwise `externalbrowser` |
| `SNOWFLAKE_PASSWORD`      | Conditional | -                 | Required if authenticator is `snowflake`, not needed for `externalbrowser` |
| `SNOWFLAKE_TOKEN`         | Conditional | -                 | Required if authenticator is `programmatic_access_token` or `oauth`         |
| `SNOWFLAKE_PRIVATE_KEY_PATH` | Conditional | -              | Path to a PEM/PKCS8 private key; for `snowflake_jwt`                       |
| `SNOWFLAKE_PRIVATE_KEY`   | Conditional | -                 | Inline private key body; alternative to `SNOWFLAKE_PRIVATE_KEY_PATH`       |
| `SNOWFLAKE_PRIVATE_KEY_PASS` | No       | -                 | Passphrase for an encrypted private key                                    |
| `SNOWFLAKE_ROLE`          | No          | -                 | Role to use for the session (uses account default if not set)              |
| `SNOWFLAKE_WAREHOUSE`     | No          | -                 | Warehouse to use (uses account default if not set)                         |
| `SNOWFLAKE_DATABASE`      | No          | -                 | Database to use (uses account default if not set)                          |
| `SNOWFLAKE_SCHEMA`        | No          | -                 | Schema to use (uses account default if not set)                            |
| `SNOWFLAKE_READONLY`      | No          | `true`            | Readonly is ON by default. Set to `false` to allow write operations        |
| `SNOWFLAKE_LOG_LEVEL`     | No          | `OFF`             | Snowflake SDK log level (`OFF`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`); logs to the OS temp dir, never the working directory |

### Finding Your Connection Settings

You can find your connection settings in Snowsight (Snowflake's web interface):

1. Sign in to [Snowsight](https://app.snowflake.com/)
2. Click on your username in the bottom-left corner to open the user menu
3. Select **Connect a tool to Snowflake**
4. Open the `Config File` tab
5. Select the Warehouse, Database, Schema you want to use
6. Copy values from the generated config file

## Available Tools

### Query Execution

#### `execute_query`

Execute a SQL query against Snowflake.

| Parameter  | Type   | Required | Default | Description                              |
| ---------- | ------ | -------- | ------- | ---------------------------------------- |
| `query`    | string | Yes      | -       | The SQL query to execute                 |
| `max_rows` | number | No       | 100     | Maximum number of rows to return         |
| `timeout`  | number | No       | -       | Query timeout in seconds                 |
| `format`   | string | No       | table   | Output format: `table`, `json`, or `csv` |

#### `explain_query`

Get the execution plan for a SQL query without running it.

| Parameter | Type   | Required | Description              |
| --------- | ------ | -------- | ------------------------ |
| `query`   | string | Yes      | The SQL query to explain |

### Connection

#### `test_connection`

Test the connection to Snowflake and return connection info including current user, role, warehouse, database, schema, and version.

### Schema Discovery

#### `list_databases`

List all accessible databases in Snowflake.

#### `list_schemas`

List all schemas in a database.

| Parameter  | Type   | Required | Description                                   |
| ---------- | ------ | -------- | --------------------------------------------- |
| `database` | string | No       | Database name (uses current if not specified) |

#### `list_tables`

List all tables in a schema.

| Parameter  | Type   | Required | Description   |
| ---------- | ------ | -------- | ------------- |
| `database` | string | No       | Database name |
| `schema`   | string | No       | Schema name   |

#### `list_views`

List all views in a schema.

| Parameter  | Type   | Required | Description   |
| ---------- | ------ | -------- | ------------- |
| `database` | string | No       | Database name |
| `schema`   | string | No       | Schema name   |

### Table Inspection

#### `describe_table`

Get detailed information about a table's structure including columns, types, and constraints.

| Parameter | Type   | Required | Description                                    |
| --------- | ------ | -------- | ---------------------------------------------- |
| `table`   | string | Yes      | Table name (can include database.schema.table) |

#### `get_table_sample`

Get a sample of rows from a table to understand its data.

| Parameter | Type   | Required | Default | Description                     |
| --------- | ------ | -------- | ------- | ------------------------------- |
| `table`   | string | Yes      | -       | Table name                      |
| `limit`   | number | No       | 5       | Number of sample rows to return |

#### `get_table_row_count`

Get the total number of rows in a table.

| Parameter | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| `table`   | string | Yes      | Table name  |

#### `get_primary_keys`

Get primary key columns for a table.

| Parameter | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| `table`   | string | Yes      | Table name  |

## MCP Resources

### `schema://current`

Returns information about the current database schema, including:
- Current database and schema names
- List of all tables
- List of all views

Access this resource to get a quick overview of the connected schema without running queries.

## MCP Prompts

### `analyze_table`

Analyze a table's structure, sample data, and get query suggestions.

| Parameter    | Type   | Required | Description          |
| ------------ | ------ | -------- | -------------------- |
| `table_name` | string | Yes      | The table to analyze |

### `explore_database`

Explore and summarize the structure of a database.

| Parameter       | Type   | Required | Description                                         |
| --------------- | ------ | -------- | --------------------------------------------------- |
| `database_name` | string | No       | Database to explore (uses current if not specified) |

### `query_builder`

Help build a SQL query based on natural language description.

| Parameter     | Type   | Required | Description                                   |
| ------------- | ------ | -------- | --------------------------------------------- |
| `description` | string | Yes      | Natural language description of desired query |
| `tables`      | string | No       | Comma-separated list of relevant tables       |

## Security Considerations

- **Readonly mode is ON by default.** The server blocks statements starting with
  write/DDL/session verbs (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `CREATE`, `ALTER`,
  `CALL`, `EXECUTE`, `USE`, `SET`, `PUT`, `REMOVE`, ...) after stripping SQL comments.
  Set `SNOWFLAKE_READONLY=false` only if you genuinely need writes.
- **Readonly mode is defense in depth, not a sandbox.** The real security boundary
  is the Snowflake role your credentials carry -- use a read-only role (and a
  dedicated PAT scoped to it) rather than relying on statement filtering.
- **Table/schema/database arguments are validated as identifiers** before being
  interpolated into SQL, so tool inputs cannot smuggle extra statements.
- **SDK logging is off by default** and never writes to the working directory or
  to stdout (which carries the MCP protocol). Set `SNOWFLAKE_LOG_LEVEL` to debug.
- **Prefer PAT or key-pair auth for agent use.** SSO (`externalbrowser`) pops a
  browser window on every fresh MCP process; tokens don't. Keep tokens in your
  MCP client config in your home directory, never in a repo.
- Never commit configuration files containing credentials to version control
- Consider using environment variables or a secrets manager for sensitive values
- The server executes queries with the permissions of the configured Snowflake user—ensure appropriate access controls are in place
- The `max_rows` parameter helps prevent accidentally returning massive result sets

## License

MIT License
