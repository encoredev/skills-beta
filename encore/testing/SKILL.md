---
name: encore-beta-testing
description: Write and run tests for an Encore v2 (beta) app with `encore test`, in TypeScript, Python, Go or Rust. Covers per-run isolated databases, buckets, topics and Redis, each service's test strategy, testing HTTP routes, mocking other services, testing subscriptions and cron handlers, config in tests, and `encore exec` for scripts.
when_to_use: >-
  User is in an Encore v2 app and wants to add, fix or run tests, test an endpoint or a database query, mock a call to another service, test a Pub/Sub handler, or understand why tests fail when run with plain `pytest`, `vitest`, `go test` or `cargo test`. Trigger phrases: "test", "encore test", "vitest", "jest", "pytest", "go test", "cargo test", "integration test", "isolated database", "mock", "supertest", "test fixture". SKIP for one-off end-to-end checks of a running app (`encore-beta-run`, `encore check`).
---

# Testing in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

**Run tests through `encore test`.** It attaches the Encore runtime to the test process and provisions infrastructure on first use, isolated from `encore run` data and from other test runs, and removes it when the process exits. Running the test tool directly fails or hits the wrong infrastructure (Python fails at `import encore`).

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`).

## Running

```bash
encore test                                # each service's `test` strategy (default per language)
encore test --services users               # service-set expression: 'users | billing', 'lang(ts)'
encore test --concurrency 1
encore test -- vitest run users.test.ts    # your own command
encore test --report-dir out --coverage    # JUnit per service and a merged coverage.lcov (not with your own command)
```

Your own command (`encore test -- <cmd>`) runs once, verbatim, from the current directory; with `--services` it runs in each selected service's directory with its environment.

Defaults: TypeScript runs the `test` script in `package.json`; Python runs `pytest` through the lockfile's package manager; Go and Rust use their language test strategy. Override with `"test": { "type": "custom", "cmd": [...] }` (or `"script"` for TypeScript) in `encore.service.json`.

## What isolation means

- Each **test process** gets a fresh database cloned from the migrated template (including rows migrations insert, never your local data), its own bucket prefix, topics and Redis namespace.
- Isolation is **per process, not per test**. Tests in one process share all of it. Use unique IDs per test, clean up in fixtures/`afterEach`, and don't assert on table-wide counts.
- Test databases run on a separate Postgres cluster with `fsync` off. Encore keeps it warm between runs.
- `appMeta().environment.type` is `"test"`. Config uses the base file plus the `test` overlay, never `*.user.config.*`. Secrets get the same values as `encore run`.

## Patterns

- **Import resources directly.** Keep declarations in modules that don't start the server, so tests can import them.
- **HTTP routes:** export the framework app from a module that doesn't listen, and test it with the framework's test client (supertest, `app.inject`, FastAPI `TestClient`, `httptest`, Axum's `oneshot`).
- **Other services don't run.** Their URLs resolve, but requests get `501 Not Implemented` with an error naming the service and path. Mock them (e.g. a custom `fetcher` for a generated TS client).
- **Pub/Sub:** topics exist so code can publish. Test a subscription by calling its handler function with a message you build. For end-to-end delivery, poll with a timeout.
- **Cron:** not scheduled in tests. Call the handler or request its route.

## Scripts against local infrastructure

`encore exec` runs any command attached to the same local databases and buckets as `encore run`, for seeds, one-off scripts and REPLs:

```bash
encore exec bun scripts/seed.ts
encore exec -- uv run python manage.py migrate
```
