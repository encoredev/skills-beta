---
name: encore-beta-run
description: Run an Encore v2 (beta) app locally and verify changes. Covers `encore check` for one-shot end-to-end checks, running in the background without blocking, `encore status`, `encore stop`, `encore logs`, `encore trace`, the service proxy and gateway addresses, the dev dashboard, and fixing services that fail to start or return 404/502.
when_to_use: >-
  User is in an Encore v2 app and wants to run, start or try it, call a route, check that a change works, see logs or traces, open the dashboard, restart or stop the app, or fix startup errors, 404s or 502s from the gateway. Trigger phrases: "run the app", "encore run", "encore check", "encore status", "does it work", "call the endpoint", "dev dashboard", "logs", "restart", "stop the app", "502", "404 from gateway", "service failed to start". SKIP for automated tests (`encore-beta-testing`) and deploys (`encore-beta-deploy`).
---

# Running an Encore v2 app locally

Commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`). Run them from the app root or any directory below it.

The daemon keeps each app open between commands. Services start on their first request and stop when idle. `encore run` attaches to that, keeps the services up, and watches the source.

## 1. Verify with `encore check` (preferred for agents)

Boots a fresh instance on free ports, runs requests, shuts down, and always exits:

```bash
encore check                                         # does it start?
encore check 'curl /users/ada -X POST -f; curl /users/1 -f'
encore check < check.txt                             # one request per line, '#' comments
```

- Paths are relative to the app's gateway. Everything after the path goes to curl verbatim.
- **Pass `-f` (or `-fsS`).** Without it, curl exits 0 on a 404 or 502, and the check "succeeds". Add `-i` to see status and headers.
- Exit 0: the app started and every command exited 0. Exit 1: read each result between `=== [i/N] ... ===` and `--- exit: N ---` on stdout; build errors and service logs are on stderr.
- The path comes right after `curl` (`curl /users -fsS`, not `curl -fsS /users`), and arguments are split like a shell's: quote JSON bodies in single quotes (`-d '{"name":"ada"}'`), or their double quotes are stripped.
- Also check bodies and side effects, not just exit codes.
- The instance is fresh, but **its data isn't**: it uses the app's local databases, so rows from earlier runs and checks are still there. Don't assert on IDs or counts; create unique data per check.

## 2. Keep it running

`encore run` takes over the terminal (a TUI). From an agent, run it in the background with plain output, then read the addresses from `encore status`:

```bash
encore run --ui plain > /tmp/encore-run.log 2>&1 &    # or --json for NDJSON
encore status                                          # gateways, service proxy, MCP URL, dashboard, problems
encore status --json
```

- `--port 4000` / `--listen` pick the base port; the default gateway gets the next port. Without them the ports vary, so always read `encore status`.
- The **service proxy** at the base address forwards `/<service>/<path>` to that service, including unexposed routes (useful for cron routes).
- `--services 'users | billing'` keeps only some services up; others start on demand.
- Quitting `encore run` detaches; the app can stay open. **For a full restart: `encore stop`, then `encore run`.** `--reset` restarts what's running in the current shell's environment.

## 3. Look at what happened

```bash
encore logs                       # latest session; app needn't be running
encore logs --errors --svc orders --since 10m
encore trace list
encore trace get <trace-id>
encore dashboard                  # local dev dashboard: traces, logs, architecture, infra
encore debug meta --format json   # what the parser found: services, APIs, resources
encore explain ENC0005            # explain an error code
```

## Fixing common failures

| Symptom | Cause and fix |
|---|---|
| `404` from the gateway | The route's `api` entry lacks `"expose": true`, or the path is outside its `prefix`. Routes keep their prefix: `"prefix": "/notes"` serves `/notes/...`. |
| `502` from the gateway | The service didn't start or isn't listening where Encore expects. Check stderr / `encore logs`. Bind to `ENCORE_LISTEN_ADDR`, not a fixed port. |
| `failed to spawn uv/bun/go/cargo` | Toolchain not installed (see `encore-beta-install`). |
| TS: `could not find the Encore runtime library ('encore-runtime.node')` under `encore test` or `encore exec` | Beta.5 only (fixed in beta.6): `export ENCORE_RUNTIME_LIB="$(brew --prefix encore-beta)/libexec/runtimes/js/encore-runtime.node"` first. |
| SDK/runtime version error at startup | Pin the SDK to `encore version` (see `encore-beta-install`). |
| Every TS service fails with `could not find ... encore-runtime.node` under `encore run`/`check`, or tests can't find the bundled Postgres, right after an upgrade | The old daemon is still running: restart it (see `encore-beta-install`). |
| Commands fail with `h2 send failed: connection error` | No daemon is running, e.g. it exited because a v1 daemon holds port 9400: start it with `ENCORE_DEVDASH_LISTEN_ADDR=127.0.0.1:9401 encore-beta daemon` (see `encore-beta-install`). |
| Parse error naming a resource | A declaration isn't at module scope, has a non-literal name/option, or a resource is aliased/passed around (see `encore-beta-code-review`). |
| Pub/Sub handler never runs | Missing start call for the language, or its module isn't imported (see `encore-beta-pubsub`). |
| Secret empty or `KeyError` | Set it with `encore secret set` (see `encore-beta-secrets`). |
| Migration failed / dirty | See `encore-beta-database`. |
| Changes not picked up | `encore stop` then run again. |

## Isolation

Each git worktree (or jj workspace) gets its own infra namespace, so parallel branches or agents don't share data. `encore namespace ls|fork|empty|delete` manages them.
