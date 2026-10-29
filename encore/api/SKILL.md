---
name: encore-beta-api
description: Write endpoints with Encore's own API framework in an Encore v2 (beta) app, for TypeScript (`@encore.dev/api`). Covers `"type": "encore-api"`, `api()` and `api.raw()`, path, query, header and cookie parameters, validation, `APIError`, middleware and auth by tags, calling other services, and testing endpoints.
when_to_use: >-
  User is in an Encore v2 app and wants Encore-style typed endpoints instead of a third-party framework, is porting a v1 Encore.ts app and wants to keep its `api()` endpoints, sees `"type": "encore-api"` or `@encore.dev/api`, or needs auth, middleware or validation for such endpoints. Trigger phrases: "api()", "@encore.dev/api", "encore-api", "Encore API framework", "typed endpoint", "APIError", "Query<>", "Header<>", "middleware", "keep my v1 endpoints". SKIP for services written with Express, Hono, Fastify, FastAPI or another framework (`encore-beta-service`), and for v1 apps (`encore.dev/api`): use the stable `encore-api` skill.
---

# Encore's API framework in v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

In v2 a service can bring its own HTTP framework, or use Encore's. With Encore's API framework you declare endpoints in code, as in v1, and Encore generates the server: no `listen`, no `api` entries in the manifest. Encore reads the endpoints' types to route, decode, validate and document requests.

It's the shortest path for a v1 Encore.ts app: endpoint code moves over almost unchanged, and the error JSON (`{"code", "message", "details"}`) is the same, so existing clients keep working.

**Read `references/<lang>.md`** for the service's language. The framework exists for TypeScript (`typescript`) and Go; only TypeScript is covered here so far. Python and Rust services use their own framework (`encore-beta-service`).

## Turning it on

Set `"type": "encore-api"` at the top of the service's manifest. Leave out `run` and `api`: Encore starts the service and learns its routes from the code.

```json
{
  "name": "users",
  "lang": "typescript",
  "type": "encore-api",
  "description": "Signs users up and manages their profiles."
}
```

## What carries over from v1, and what doesn't

| v1 (`encore.dev/api`) | v2 API framework |
|---|---|
| `api()`, `api.raw()`, `Query`, `Header`, `Cookie`, `HttpStatus`, validation types, `APIError` | Same names, from `@encore.dev/api` |
| `new Service("users")` in `encore.service.ts` | `encore.service.json` with `"type": "encore-api"`; optionally `export default new Service({ middlewares })` in the service's `index.ts` |
| `middleware()` | Same, with `target: { expose, isRaw, tags }` |
| `authHandler`, `Gateway`, `auth: true`, `getAuthData()` | None yet. Use middleware on tagged endpoints (see the reference) |
| `~encore/clients` | None yet: no clients are generated between these services. Call `serviceUrl()` with `fetch` |
| `api.static` | None. Serve files from a raw catch-all endpoint |
| `api.streamIn` / `streamOut` / `streamInOut` | None yet |

## Rules the router enforces

- **Parameter names in the same position:** from 2.0.0-beta.6, `/wines/:id/notes` and `/wines/:from/merge` can coexist, and each route keeps its own names. On beta.5 they can't: the app parses, then the service fails at startup and the gateway answers 502. The same route twice under one method is refused in every version.
- A wildcard is the last segment, `{*rest}`, and doesn't match the bare prefix: `/{*path}` doesn't take `/`, so add a `/` endpoint too.
- More specific routes win, also across services: one service's `/{*path}` catch-all doesn't shadow another service's `/users/:id`.
- Endpoints are private unless `expose: true`. Through the gateway a private route answers 405 (or falls through to a public catch-all); the service proxy (`encore run`) still reaches it.

## Verify

```bash
encore debug meta --format json   # each service's "encore" API spec: endpoints, paths, request/response types
encore check 'curl /users/1 -f'
```
