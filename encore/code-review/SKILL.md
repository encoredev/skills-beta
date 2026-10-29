---
name: encore-beta-code-review
description: Review Encore v2 (beta) code in TypeScript, Python, Go or Rust for the mistakes that break static discovery, permissions, startup or correctness. A checklist covering resource declarations, handles, manifests, listening, subscriptions, SQL, secrets, exposure and leftover v1 APIs.
when_to_use: >-
  User is reviewing a PR or auditing code in an Encore v2 app, or a change parses or deploys but misbehaves. Also use as a self-check before declaring Encore v2 work done. Trigger phrases: "review", "audit", "PR", "before merge", "anti-patterns", "best practices", "check my Encore code". SKIP for v1 apps.
---

# Reviewing Encore v2 code

Run `encore debug meta --format json` (as `encore-beta` unless `encore version` reports 2.x) to see what the parser found, and `encore check` with `curl -f` requests for behavior. Then go through the list. Language details are in each topic skill's `references/<lang>.md`.

## Discovery (parse errors or missing resources)

- [ ] Every resource (database, topic, subscription, bucket, Redis, cron job, secret, config, metric) is declared at **module/package scope**, never inside a function, class, conditional or loop.
- [ ] Names and options are **literals**. Computed subscription options are silently ignored.
- [ ] Resources aren't aliased (`other = db`), stored-method (`q = db.query`), returned or passed to functions. Helpers get a **handle/reference** (`db.read()`, `bucket.ref<Uploader>()`, `sessions.write()`, `metric.ref()`).
- [ ] Redis commands are called on their group in one expression (`cache.string.get(...)`).
- [ ] Resources from another service are bound by name (`Topic.named(...)`), not imported from that service's internals.
- [ ] Declarations live in modules the service imports at startup (subscriptions especially). An API framework service's generated entry imports only `index.ts` and modules that declare endpoints.
- [ ] TS subscription callbacks are properties (`decode: (d) => ...`), not method shorthand.
- [ ] Python `Secret[str]("X")`/`Config[T]("x")` carry the type argument on the constructor.

## Manifests and startup

- [ ] `encore.app.json` (not `encore.app`) at the root; each service has a strict-JSON `encore.service.json` with `name`, `lang`, and a `run` strategy that matches the real start command.
- [ ] The server binds to `ENCORE_LISTEN_ADDR`, with no hardcoded port.
- [ ] Routes include the service's `prefix`.
- [ ] SDK version pinned to the CLI version.
- [ ] Subscriptions are started: Python `encore.run(app)` or `await encore.start()`, Go `encore.Start()`, Rust `#[encore::pubsub::register]` + `encore::initialize()`.
- [ ] Graceful shutdown kept: TS `onShutdown` drains the server; Python servers other than `encore.run` call `shutdown.begin()`/`completed()`; no forking after `import encore`.

## Exposure and security

- [ ] `"expose": true` only where routes are meant to be public, with the framework's auth in front. `prefix: "/"` + `expose` publishes everything, including cron routes.
- [ ] Untrusted input is validated (Zod, Pydantic, ...); type annotations alone don't validate. Pub/Sub payloads validated with a decoder where it matters.
- [ ] No secret values in code, config files, logs or committed `.secrets.user.toml`. Secrets needed in deployed environments are shared/global (beta limitation).
- [ ] Sensitive endpoints marked `"sensitive": true` (in the API framework, `sensitive: true` on `api()`), including private ones that receive cookies or tokens.
- [ ] CORS matches what the frontend needs: from 2.0.0-beta.6 the default gateway allows credentialed requests from no other origin, so a frontend on another origin needs `allow_origins_with_credentials` in `encore.gateway.json` (snake_case keys). No `{"unsafe": "*"}` without a reason. On beta.5 the default reflects any `Origin` with credentials: restrict it (see `encore-beta-service`).
- [ ] Raw endpoints (API framework `api.raw`) set a `bodyLimit` that fits what they take (2 MiB by default, enforced on raw endpoints from 2.0.0-beta.6; on beta.5 they must cap the body themselves).

## Correctness

- [ ] SQL is parameterized (tagged templates / `$1` params). `raw` variants interpolate only trusted identifiers.
- [ ] TS transactions call `commit()` explicitly; `await using` rolls back otherwise.
- [ ] Python cursors are consumed once; early exits use `async with`.
- [ ] Migrations are new files, never edits to applied ones, and are compatible with the previous release.
- [ ] Pub/Sub handlers read `message.data` and are **idempotent**. Publishes are awaited / errors checked.
- [ ] Cron handlers are idempotent, `POST`, and their path matches the job literally.
- [ ] Redis cache entries have TTLs; nothing irreplaceable lives in a non-persistent database.
- [ ] Logs use constant messages with fields, log errors with the error object, and never include credentials.
- [ ] Metric labels are low-cardinality.
- [ ] Generated files (`encore_gen/`, `.encore/`) aren't edited or committed.

## Leftover v1 APIs

- [ ] No `encore.dev/...` imports in TypeScript (v2 is `@encore.dev/sdk/...`), no `authHandler`, `~encore/clients`, `encore.service.ts`, `SQLDatabase`, `CacheCluster`, `CronJob({ endpoint })`, `.secrets.local.cue`. `api(...)`, `APIError`, `Query<>` and `middleware()` are fine from `@encore.dev/api` in a service with `"type": "encore-api"`, and nowhere else.
- [ ] No Go `//encore:api` annotations or `encore.dev` imports (v2 is `sdk.encore.dev`).
