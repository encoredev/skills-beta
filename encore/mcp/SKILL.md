---
name: encore-beta-mcp
description: Connect an AI coding tool (Claude Code, Cursor, VS Code, Codex, Zed) to the Encore v2 (beta) daemon's local MCP server over Streamable HTTP, and use its tools to inspect and drive local apps: status, calling routes, traces, logs, databases, Pub/Sub, buckets, Redis, secrets, config, cron jobs and tests.
when_to_use: >-
  User wants their AI tool to talk to Encore v2 through MCP, asks for the Encore MCP URL, or an MCP setup from v1 (`encore mcp run --app=...`) doesn't work with the beta. Trigger phrases: "MCP", "Encore MCP", "connect Claude Code to Encore", "Cursor MCP", "encore mcp", "mcp clients", "127.0.0.1:9400/mcp". SKIP for v1 (`encore mcp start`/`run` over stdio).
---

# The Encore v2 local MCP server

Commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

In v2, the Encore daemon itself serves MCP over **Streamable HTTP** at `/mcp` on its HTTP address. There is **no `encore mcp run`/`start`** (those were v1's stdio bridge). One endpoint serves every app on the machine; each tool takes an `app` argument, the absolute path of the app's directory or any directory in it.

## Find the URL

```bash
encore status          # prints a line like:  mcp  http://127.0.0.1:9450/mcp
```

The port varies by machine (often 9400, but not always), so read the URL from `encore status` and use it in place of the examples below.

## Connect

The endpoint only accepts loopback connections and needs no auth, so the config is just the URL.

Claude Code:

```bash
claude mcp add --transport http encore http://127.0.0.1:9400/mcp
```

Cursor (`.cursor/mcp.json`) and other tools that take a URL:

```json
{ "mcpServers": { "encore": { "url": "http://127.0.0.1:9400/mcp" } } }
```

VS Code (`.vscode/mcp.json`):

```json
{ "servers": { "encore": { "type": "http", "url": "http://127.0.0.1:9400/mcp" } } }
```

The daemon must be running: any `encore` command starts it. `encore llm-rules init` doesn't write MCP configuration.

## Tools

| Area | Tools |
|---|---|
| App | `status`, `metadata`, `services_list`, `rebuild`, `sessions_list`, `sessions_stop` |
| Requests | `call` (send a request to the app) |
| Tests and jobs | `test`, `job`, `cron_run` |
| Telemetry | `traces_list`, `traces_get`, `logs`, `metrics_list`, `metrics_get` |
| Postgres | `postgres_query`, `postgres_execute`, `postgres_migrations`, `postgres_migrate`, `postgres_reset` |
| Pub/Sub | `pubsub_queues`, `pubsub_history`, `pubsub_message`, `pubsub_publish`, `pubsub_pause`, `pubsub_resume`, `pubsub_discard`, `pubsub_release` |
| Object storage | `objects_buckets`, `objects_list`, `objects_read`, `objects_write`, `objects_delete`, `objects_search` |
| Redis | `redis_keys`, `redis_get`, `redis_delete` |
| Secrets and config | `secrets_list`, `secrets_set`, `config_list`, `config_show`, `config_schema`, `config_set`, `config_validate` |
| Services | `service_templates`, `service_create` |
| Namespaces | `namespace_list`, `namespace_fork`, `namespace_empty` |

Use MCP only if the tools are actually available in the session; otherwise the CLI covers the same ground (`encore-beta-run`).

```bash
encore mcp clients     # which MCP clients have used apps on this machine
```
