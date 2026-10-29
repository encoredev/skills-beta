# Encore Skills (Beta): agent instructions

This repository holds agent skills for **Encore v2**, in beta. Skills live in `encore/<topic>/SKILL.md`, with per-language API references in `encore/<topic>/references/{typescript,python,go,rust}.md`. `plugins/encore-skills-beta/skills` is a symlink to `encore/`.

## Writing and editing skills

- **One skill per topic, not per language.** Put what's true in every language in `SKILL.md`, and language APIs in `references/<lang>.md`. Every topic `SKILL.md` tells the agent to read the reference for the service's `lang`.
- Skill names are `encore-beta-<topic>`, so they can be installed beside the stable `encoredev/skills`.
- Frontmatter: `name`, `description` (what it covers), `when_to_use` (triggers, trigger phrases, and a SKIP line).
- Commands are written `encore ...`. The beta CLI is installed as `encore-beta`, so every skill says to use `encore-beta` unless `encore version` reports 2.x.
- Verify every API and flag against the CLI (`encore-beta <cmd> --help`) and the SDK source in `encoredev/v2-encore` before writing it down. The beta docs lag the CLI in places (see below).
- Keep references compact and code-first. When a language lacks a feature, say so in one line.
- When adding or removing a skill, update `.claude-plugin/marketplace.json`, `README.md` and the table below.
- Style: no em dashes, and don't use the word "shape".

## Known places the beta docs disagree with the CLI

| Docs say | CLI/SDK actually |
|---|---|
| Python: `encore.app` | `encore.app.json` (v2 refuses `encore.app`) |
| Python: `.secrets.local.toml` | `.secrets.user.toml` |
| TS: config env types `ephemeral` | Config overlays use `local`, `test`, `development`, `preview`, `production` (app metadata still reports `ephemeral`) |
| TS: `encore db shell <database>` | `encore db shell --database <database>` (all `encore db` commands) |
| Python: `encore app new --template python` | No `--template` on `app new`; use `encore service new --template ...` |
| Python: uvicorn + `encore.start()` | Templates use `encore.run(app)`, which also starts subscriptions and handles shutdown |
| Python: `cursor.one()` returns `None` when no row | Raises `EncoreError`; use `first()` for an optional row |
| Python: numeric/timestamps come back as `str`, no `datetime` params | `Decimal`, `datetime`, `date`, `time`, both ways |
| MCP at `127.0.0.1:9400/mcp` | Port varies; read it from `encore status` |

## Known CLI issues the skills work around (2.0.0-beta.7)

The skills describe the latest beta only. When an issue is fixed, delete its workaround from the skills and from this list.

- Encore Cloud builds don't install npm packages for TypeScript services, so TS apps can't deploy (API framework services fail with `no node_modules directory at or above`). Noted in `deploy`.
- Upgrading the CLI leaves the old daemon running from the removed version's directory, and `encore version` prints the daemon's version. Tests then can't find the bundled Postgres and TS services can't find `encore-runtime.node`. No restart command; workaround in `install` and `run`.
- The v2 daemon's dev dashboard wants 127.0.0.1:9400, like v1's; with a v1 daemon running it can exit silently. Workaround: `ENCORE_DEVDASH_LISTEN_ADDR`.
- Encore Cloud stores app-wide secrets only.
- `@encore.dev/api`: `{*rest}` doesn't match the bare prefix; calling an endpoint directly (tests) skips request validation and middleware.
- No clients are generated between `"type": "encore-api"` services (`dependencies` + `encore gen clients` writes nothing), and there's no browser client generation for them.
- `encore test` gives each test process its own database copies and drops them on exit; Vitest's default `forks` pool (a process per file) backs up the test Postgres. Skills recommend `pool: "threads"`.
- Rust: a resource declared inside an inline `mod { use ...; static X = Topic::new(..) }` block whose type is only imported inside that block is silently dropped from discovery. Workaround in the Rust service reference.
- Python: `encore.run(app)` is the recommended entry point (templates use it) but the beta docs don't mention it.
- `encore check` passes curl arguments verbatim, so a request without `-f` "succeeds" on a 404 or 502.

## Skills

| Skill | Trigger keywords |
|-------|------------------|
| `encore-beta-getting-started` | try Encore v2, beta, get started, hello world |
| `encore-beta-install` | install, encore-beta, brew, command not found, version mismatch, failed to spawn |
| `encore-beta-login` | log in, auth key, ENCORE_AUTH_KEY, whoami, CI |
| `encore-beta-create-app` | encore app new, app init, service new, template, llm-rules |
| `encore-beta-service` | encore.service.json, expose, ENCORE_LISTEN_ADDR, serviceUrl, base_url, encore.run, generated clients, gateway CORS, build config |
| `encore-beta-api` | Encore's API framework, @encore.dev/api, encore-api, api(), APIError, middleware, keep v1 endpoints |
| `encore-beta-run` | encore run, encore check, status, stop, logs, traces, 404, 502 |
| `encore-beta-mcp` | MCP, connect Claude Code/Cursor, mcp clients |
| `encore-beta-deploy` | deploy, platform link, git push encore, platform deploy, build docker |
| `encore-beta-migrate-from-v1` | v1 to v2, encore.service.ts, ~encore/clients, encore migrate |
| `encore-beta-code-review` | review, audit, PR, anti-patterns |
| `encore-beta-database` | Postgres, migration, query, transaction, ORM, encore db |
| `encore-beta-pubsub` | topic, subscription, publish, event, ordered, dead letter |
| `encore-beta-bucket` | bucket, upload, S3, signed URL, public URL |
| `encore-beta-redis` | Redis, cache, TTL, rate limit, leaderboard |
| `encore-beta-cron` | cron, schedule, daily, hourly, periodic |
| `encore-beta-secrets` | secret, API key, encore secret set, .secrets.user.toml |
| `encore-beta-config` | config, feature flag, per environment, encore config set |
| `encore-beta-observability` | logging, metrics, tracing, OpenTelemetry |
| `encore-beta-testing` | encore test, vitest, pytest, go test, cargo test, isolation |
