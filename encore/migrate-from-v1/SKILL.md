---
name: encore-beta-migrate-from-v1
description: Move an Encore v1 app (Encore.ts or Encore.go) to Encore v2 (beta). Covers the file and package changes, keeping `api()` endpoints on Encore's API framework or porting them to another framework, replacing `~encore/clients`, mapping each infrastructure API, and verifying with `encore check`. Also covers `encore migrate` for driving the migration with a coding agent.
when_to_use: >-
  User wants to try an existing Encore v1 app on the v2 beta, keep or port `api()` endpoints or `//encore:api` handlers (Encore's API framework, Express/Hono/Fastify/net/http), replace `~encore/clients`, or map a v1 API (SQLDatabase, CacheCluster, CronJob endpoint, secrets) to v2. Also for `encore migrate`. Trigger phrases: "migrate to v2", "upgrade to Encore v2", "port to the beta", "v1 to v2", "encore.service.ts to encore.service.json", "replace ~encore/clients", "encore migrate". SKIP for migrating a non-Encore backend into Encore v1 (use the stable `encore-migrate`).
---

# Migrating from Encore v1 to v2

Commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

**v2 is a beta. There's no need to migrate production apps yet; v1 stays fully supported.** Work on a branch, and keep the v1 CLI for the main line.

The biggest change is the HTTP layer. Each v2 service is its own process, and either uses a framework of your choice or **Encore's API framework** (`"type": "encore-api"`, TypeScript and Go). The infrastructure APIs carry over with mostly the same design under a new package.

**For a TypeScript v1 app, start with Encore's API framework** (`encore-beta-api`): `api()`, `api.raw()`, `Query`/`Header`, validation types, middleware and `APIError` keep working from `@encore.dev/api`, so endpoint code barely changes and existing clients keep working. What you rebuild: auth (middleware on tagged endpoints), service-to-service calls (HTTP to the base URL), `api.static` (a raw catch-all) and tests that called other services. Porting to Express, Hono or Fastify also works, but means rewriting every endpoint, its validation and its errors.

**Read `references/<lang>.md`** for the language being migrated (`typescript` for Encore.ts, `go` for Encore.go) for the API-by-API mapping.

## File and structure changes

| v1 | v2 |
|---|---|
| `encore.app` | `encore.app.json` (rename it; v2 refuses `encore.app`) |
| `encore.service.ts` / Go service package | `encore.service.json` with `name`, `lang`, `run`, `api` |
| One process for all services (or Encore-managed process allocation) | One process per service, started with your command |
| Encore's API framework | Encore's API framework (`"type": "encore-api"`, no `run`/`api` entries), or any framework that Encore routes to via `api` entries |
| `~encore/clients` / direct Go service calls | HTTP via the service's base URL, or clients generated from an OpenAPI/GraphQL/gRPC spec |
| `.secrets.local.cue` | `.secrets.user.toml` |
| `secret("X")` app-wide | `secret("X")` is per service; `sharedSecret("X")` for app-wide |

Existing migration files work as they are.

## Steps

1. `brew install encoredev/tap/encore-beta` and create a branch.
2. Rename `encore.app` to `encore.app.json`.
3. For each service, choose Encore's API framework (keep the `api()` endpoints; see `encore-beta-api`) or a framework of your own (Express, Hono and Fastify map closely to `api()`; Go uses `net/http` or a router, listening on `ENCORE_LISTEN_ADDR`).
4. Replace `encore.service.ts` with `encore.service.json`: `"type": "encore-api"` for Encore's framework, otherwise a `run` command and an `api` entry for its routes (`"expose": true` for what v1 had `expose: true` on; routes keep their prefix).
5. Switch imports to the v2 SDK (see the reference) and pin it to the CLI version.
6. Port auth handlers to middleware. With your own framework, also port `APIError` to its error responses and validation types to Zod or equivalent.
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
