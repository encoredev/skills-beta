---
name: encore-beta-getting-started
description: Set up Encore v2 (beta) from scratch in TypeScript, Python, Go or Rust. Install the beta CLI, create an app, add a service from a template, expose it, and verify it runs, with a pointer to the skill for each step. Ends where building starts.
when_to_use: >-
  User has no Encore v2 app yet and wants to try the Encore beta / Encore v2 end to end. For a single step use `encore-beta-install`, `encore-beta-create-app`, `encore-beta-run` or `encore-beta-mcp`. SKIP if the user already has a v2 app and asks about a specific feature. Trigger phrases: "try Encore v2", "Encore beta", "get started with the beta", "set up Encore v2", "hello world v2".
---

# Getting started with Encore v2 (beta)

Encore v2 separates the HTTP layer from the infrastructure: each service is an ordinary server in the framework you choose (Express, Hono, Fastify, Next.js, FastAPI, Django, Flask, net/http, Axum, ...), and Encore's Infra SDK declares the databases, Pub/Sub, buckets, Redis, cron jobs, secrets and config it needs. TypeScript, Python, Go and Rust services can live in one app.

Encore v2 is a beta and its APIs can change. For production apps on the stable version, use the v1 Encore skills.

## 1. Install

```bash
brew install encoredev/tap/encore-beta
encore-beta version
```

It installs as `encore-beta` next to v1. Install the toolchain the service will use too (Bun for the TS templates, `uv` for Python, Go, Cargo). Details: `encore-beta-install`.

## 2. Create an app and a service

```bash
encore-beta app new my-app --llm-rules none
cd my-app
encore-beta service new --ui plain                                        # list templates
encore-beta service new api --template typescript-hello-world --ui plain  # or python-fastapi, go-hello-world, rust-axum, ...
```

Details: `encore-beta-create-app`.

## 3. Expose it

Templates are private by default. In `api/encore.service.json`, add `"expose": true` to the `api` entry:

```json
"api": [{ "type": "http", "prefix": "/api", "name": "api", "expose": true }]
```

## 4. Verify

```bash
encore-beta check 'curl /api/hello -fsS'
```

Use a path under the `prefix` in the service's `encore.service.json`. Exit 0 with the expected body means it works. To keep it running: `encore-beta run --ui plain` in the background, then `encore-beta status` for the addresses and dashboard. Details: `encore-beta-run`.

## 5. Optional

- Connect your AI tool to the daemon's MCP server: `encore-beta-mcp`.
- Log in for Encore Cloud: `encore-beta-login`, then `encore-beta-deploy`.

## Where to go next

| To... | Skill |
|---|---|
| Configure services, routing, service calls | `encore-beta-service` |
| Postgres | `encore-beta-database` |
| Pub/Sub | `encore-beta-pubsub` |
| Files | `encore-beta-bucket` |
| Redis / caching | `encore-beta-redis` |
| Scheduled jobs | `encore-beta-cron` |
| API keys | `encore-beta-secrets` |
| Settings per environment | `encore-beta-config` |
| Logs, metrics, tracing | `encore-beta-observability` |
| Tests | `encore-beta-testing` |
| Move a v1 app | `encore-beta-migrate-from-v1` |
