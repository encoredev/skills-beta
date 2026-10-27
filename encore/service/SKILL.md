---
name: encore-beta-service
description: Add or configure a service in an Encore v2 (beta) app with `encore.service.json`, in TypeScript, Python, Go or Rust. Covers the run, build and test strategies, the `api` entries and `expose`, listening on `ENCORE_LISTEN_ADDR`, startup and graceful shutdown, calling other services, and generated clients.
when_to_use: >-
  User is working in an Encore v2 app (has `encore.app.json`, `encore.service.json`, `@encore.dev/sdk`, `encore-dev`, `sdk.encore.dev` or the `encore` Rust crate) and wants to add a service, wire an existing Express/Hono/Fastify/Next.js/FastAPI/Django/Flask/net/http/Axum server into Encore, make routes public, call another service, or fix a service that won't start or isn't reachable through the gateway. Trigger phrases: "add a service", "encore.service.json", "expose", "ENCORE_LISTEN_ADDR", "serviceUrl", "base_url", "call another service", "gateway returns 404", "generated client", "dependencies", "encore.run". SKIP for v1 apps (`encore.app` + `encore.service.ts`/`//encore:api`): use the stable Encore skills.
---

# Encore v2 services

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

In Encore v2, each service is an ordinary server in the framework of your choice; Encore starts it, routes to it, provisions what it declares, and deploys it. Keep the app's existing framework, routes and conventions. TypeScript and Go services can instead use Encore's own API framework (`"type": "encore-api"`, see `encore-beta-api`), where Encore generates the server from `api()` declarations, as in v1.

After reading this, **read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`: the `lang` field in its manifest). It covers installing the SDK at the right version, startup, shutdown and service calls in that language.

## The two manifests

- `encore.app.json` at the app root (HuJSON: comments and trailing commas allowed; `{}` is valid). v2 refuses a v1 `encore.app` file.
- `encore.service.json` (strict JSON) in each service directory. Every file under that directory belongs to the service. A single-service app can keep it at the root.

```json
{
  "name": "users",
  "lang": "typescript",
  "description": "Signs users up and manages their profiles.",
  "run": { "type": "custom", "cmd": ["bun", "run", "index.ts"] },
  "api": [{ "type": "http", "prefix": "/users", "expose": true }]
}
```

| Field | Notes |
|---|---|
| `name` | Unique in the app. |
| `lang` | `typescript`, `python`, `go` or `rust`. Selects the default `run`/`test`/`build`. |
| `type` | `"encore-api"` for a service written with Encore's API framework (`encore-beta-api`); then leave out `run` and `api`. Absent otherwise. |
| `run` | How `encore run` starts it: `custom` (`cmd`, optional `build`), `script` (a `package.json` script), or the language default (`python`, `go`, `rust`). See the reference. |
| `test` | How `encore test` runs its tests. Same strategy types. |
| `build` | How it's built for deploy. TS default bundles with Rolldown (`entrypoints`, `external`, `hoist`). |
| `api` | What it serves (below). |
| `dependencies` | Services it calls through generated clients. |
| `docker` | `base_image`, `entrypoint` (required for `script`/`custom` builds). |
| `frontend` | Marks a service hosting a web frontend such as Next.js. |

Each service is its own process and package, with its own dependencies and lockfile. Services in one app can use different languages and share infrastructure such as Pub/Sub topics.

## Listening

Encore passes the address in `ENCORE_LISTEN_ADDR` as `host:port`. **Bind to it; never hardcode a port.** Falling back to a fixed address when it's unset keeps the service runnable outside Encore.

## `api` entries and exposure

```json
"api": [
  { "type": "http", "prefix": "/webhooks", "expose": true, "methods": ["POST"] },
  { "type": "http", "prefix": "/", "name": "web", "expose": true },
  { "type": "openapi", "spec": "./openapi.yaml" },
  { "type": "graphql", "schema": "schema.graphql", "expose": true },
  { "type": "grpc", "protos": "proto" }
]
```

- `http`: everything at or below `prefix`, **forwarded with the prefix included**. A service with `"prefix": "/notes"` serves routes like `/notes/123`, not `/123`. `name` defaults to the prefix's last segment, so `"prefix": "/"` needs an explicit `name` (an identifier: `[A-Za-z_][A-Za-z0-9_]*`).
- `openapi`/`graphql`/`grpc`: Encore learns the endpoints, names them in traces, and can generate clients.
- **Endpoints are private unless exposed.** Without `"expose": true` the gateway answers 404 for them, though other services can still call them. `true` means the default gateway (`api`); a list such as `["admin"]` names gateways. OpenAPI marks exposure per operation with `x-encore: { expose: true }`; gRPC with the `(encore.expose)` option.
- `"sensitive": true` keeps headers and bodies out of deployed traces.
- `expose: true` on `"prefix": "/"` makes every route public once deployed, including cron routes. Add auth with the framework's middleware, or keep internal routes under an unexposed prefix.

Auth, validation, middleware and error responses for your routes are the framework's job in v2.

## Gateway and CORS

The default gateway, `api`, needs no manifest. **By default it allows any origin, with credentials** (it reflects the request's `Origin`). To restrict it, as v1's `global_cors` did, add a gateway manifest **in a directory of its own**: a gateway owns its directory exclusively, so at the app root it overlaps every service (ENC0020). It's strict JSON: no comments, unlike `encore.app.json`. For example `gateway/encore.gateway.json`:

```json
{
  "name": "api",
  "default": true,
  "cors": {
    "allow-origins-with-credentials": ["https://example.com"],
    "allow-origins-without-credentials": ["https://example.com"]
  }
}
```

Services can't read the gateway's public URL: `appMeta().gateways` is empty inside services in beta.5. If links or sign-in need it, keep it in config with per-environment overlays (`encore-beta-config`).

## Declaring infrastructure (all languages)

Encore finds resources by parsing source, so:

- Declare them at module/package scope, with literal names and options, in a module the service imports at startup.
- Resources belong to the service whose directory holds the declaration. Encore grants each service only the access its code uses.
- Use a resource by calling methods on its declared variable (or an import of it). Don't alias it, store its methods or pass it to a function; pass a handle such as `db.read()` or a typed reference instead.
- To use a resource another service declares, possibly in another language, bind it by name (`Topic.named(...)`, `Database.named(...)` and equivalents) rather than importing that service's code.

## Calling other services

Plain HTTP (or gRPC) to the other service's base URL, which Encore resolves per environment and which reaches unexposed endpoints too: `serviceUrl("orders")` in TypeScript, `base_url("orders")` in Python, and the equivalents in Go and Rust (see the reference). Propagate the W3C `traceparent` header to join traces; HTTP client instrumentation does this.

If the target declares an `openapi`, `graphql` or `grpc` API, list it in the caller's `"dependencies"`. `encore run` keeps a typed client in the caller's `encore_gen/clients/` directory (`encore gen clients` regenerates). Don't edit `encore_gen/` or `.encore/`.

## Adding a service

Prefer a template; it writes the manifest, pins the SDK and installs dependencies:

```bash
encore service new --ui plain                          # list templates (non-interactive)
encore service new api --template typescript-fastify --ui plain
```

See the `encore-beta-create-app` skill for the template list. By hand, write `encore.service.json`, add the SDK at the CLI's version, and bind to `ENCORE_LISTEN_ADDR`.

## Inspect

```bash
encore service ls
encore status                       # what's running, where, and what's wrong
encore debug meta --format json     # every service, API and resource the parser found
```

Then verify with `encore check` (see `encore-beta-run`), using `curl -f` so a 404 fails the check.
