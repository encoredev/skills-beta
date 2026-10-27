---
name: encore-beta-migrate-from-v1
description: Move an Encore v1 app (Encore.ts or Encore.go) to Encore v2 (beta). Covers the file and package changes, porting `api()` endpoints to a framework, replacing `~encore/clients`, mapping each infrastructure API, and verifying with `encore check`. Also covers `encore migrate` for driving the migration with a coding agent.
when_to_use: >-
  User wants to try an existing Encore v1 app on the v2 beta, port `api()` endpoints or `//encore:api` handlers to Express/Hono/Fastify/net/http, replace `~encore/clients`, or map a v1 API (SQLDatabase, CacheCluster, CronJob endpoint, secrets) to v2. Also for `encore migrate`. Trigger phrases: "migrate to v2", "upgrade to Encore v2", "port to the beta", "v1 to v2", "encore.service.ts to encore.service.json", "replace ~encore/clients", "encore migrate". SKIP for migrating a non-Encore backend into Encore v1 (use the stable `encore-migrate`).
---

# Migrating from Encore v1 to v2

Commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

**v2 is a beta. There's no need to migrate production apps yet; v1 stays fully supported.** Work on a branch, and keep the v1 CLI for the main line.

The biggest change: **Encore no longer provides the HTTP layer.** v1's `api()` endpoints, request validation, auth handlers, middleware and errors become your framework's. The infrastructure APIs carry over with mostly the same design under a new package.

**Read `references/<lang>.md`** for the language being migrated (`typescript` for Encore.ts, `go` for Encore.go) for the API-by-API mapping.

## File and structure changes

| v1 | v2 |
|---|---|
| `encore.app` | `encore.app.json` (rename it; v2 refuses `encore.app`) |
| `encore.service.ts` / Go service package | `encore.service.json` with `name`, `lang`, `run`, `api` |
| One process for all services (or Encore-managed process allocation) | One process per service, started with your command |
| Encore's API framework | Any framework; Encore routes to it via `api` entries |
| `~encore/clients` / direct Go service calls | HTTP via the service's base URL, or clients generated from an OpenAPI/GraphQL/gRPC spec |
| `.secrets.local.cue` | `.secrets.user.toml` |
| `secret("X")` app-wide | `secret("X")` is per service; `sharedSecret("X")` for app-wide |

Existing migration files work as they are.

## Steps

1. `brew install encoredev/tap/encore-beta` and create a branch.
2. Rename `encore.app` to `encore.app.json`.
3. For each service, pick a framework and port its endpoints. Express, Hono and Fastify map closely to `api()`; Go uses `net/http` or a router. Have the server listen on `ENCORE_LISTEN_ADDR`.
4. Replace `encore.service.ts` with `encore.service.json`: a `run` command and an `api` entry for its routes (`"expose": true` for what v1 had `expose: true` on). Routes keep their prefix.
5. Switch imports to the v2 SDK (see the reference) and pin it to the CLI version.
6. Port auth handlers to framework middleware, `APIError` to framework error responses, and validation types to Zod or equivalent.
7. Replace service-to-service calls with HTTP to the service's base URL, or generated clients if the target has a spec.
8. Run `encore check` with `curl -f` requests for every route that used to exist, and `encore test`.

## `encore migrate`

`encore migrate` drives a coding agent (any ACP agent: Claude Code, Codex, Gemini CLI, ...) through the migration from the dev dashboard. It finds services and what they use, proposes phases, migrates each phase's services side by side in git worktrees, checks them, merges them, and asks you when a choice is yours. Progress lives in `.encore/migration/`.

```bash
encore app init           # if the checkout isn't an app yet
encore migrate            # opens it in the dashboard
encore migrate status
encore migrate answer ...
encore migrate pause | resume | complete
```
