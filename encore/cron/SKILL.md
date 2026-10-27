---
name: encore-beta-cron
description: Schedule recurring work in an Encore v2 (beta) app with cron jobs, in TypeScript, Python, Go or Rust. A job is a scheduled POST to a path on the declaring service. Covers `every` intervals, cron expressions, serving the route, keeping it private, and running jobs locally.
when_to_use: >-
  User is in an Encore v2 app and wants something to run on a schedule: hourly, daily, nightly, every N minutes, at a time in UTC, a cleanup or aggregation job. Also use when a cron job doesn't run locally. Trigger phrases: "cron", "schedule", "every hour", "daily at", "nightly", "periodic", "recurring", "batch job", "CronJob", "cron.Job". SKIP for v1 apps (`endpoint:` in the CronJob).
---

# Cron jobs in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

A cron job is a scheduled HTTP request. When it's due, Encore sends a `POST` for the job's `path` to the service that declares it. You serve that path with your framework like any other route. v1 took an `endpoint`; v2 takes a `path`.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`).

## Rules for every language

- Declare at module/package scope inside the service that serves the path. Name and every option are literals. Names are unique kebab-case IDs; keep the name if you move the declaration.
- `path` starts with `/` and must be written literally in both the job and the route.
- Exactly one of:
  - `every`: a duration (`"10m"`, `"6h"`). Runs are laid out from midnight UTC, so the interval must divide 24 hours evenly, in whole minutes, at most 24h (`"7h"` and `"50m"` are rejected).
  - `schedule`: a five-field cron expression in UTC (`"0 4 * * *"`). No `@daily` shorthands.
- **Make the handler idempotent.** A job can run more than once.
- **Keep the route private.** Encore calls the service directly, so the route needn't be on a gateway. If the service exposes `"prefix": "/"`, the cron route is public too: put it under an unexposed prefix (e.g. a second `http` entry for `/internal` without `expose`) or check the caller.
- Keep runs short. For long work, publish to a Pub/Sub topic from the job and do the work in a subscription, which gets retries.

## Running locally

Cron jobs are **not scheduled** under `encore run` or `encore test`. Trigger one yourself through the local service proxy, which `encore run` serves at its base address and which forwards `/<service>/<path>` to the service:

```bash
encore status                                            # shows the base address
curl -X POST http://localhost:<base-port>/sessions/internal/cleanup
```

An agent connected to the local MCP server can run a job by name with the `cron_run` tool. In tests, call the handler or request the route with the framework's test client.

Deployed runs show in the Encore Cloud dashboard.
