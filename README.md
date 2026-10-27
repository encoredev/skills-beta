# Encore Skills (Beta)

Agent skills for building backend applications with **Encore v2**, currently in beta, in TypeScript, Python, Go and Rust.

> Encore v2 is in beta and its APIs can change before the stable release. For the stable version of Encore, use [encoredev/skills](https://github.com/encoredev/skills). These skills are named `encore-beta-*`, so both sets can be installed at once.

## What's different in v2

Encore v2 separates the HTTP layer from the infrastructure. Each service is an ordinary server in the framework you choose (Express, Hono, Fastify, Next.js, FastAPI, Django, Flask, `net/http`, Axum, ...), and Encore's Infra SDK declares what it needs: Postgres, Pub/Sub, object storage, Redis, cron jobs, secrets, config and metrics. Services in one app can be written in different languages and share infrastructure.

- **Local:** `encore run` starts real local infrastructure (bundled Postgres, a Pub/Sub broker, object storage and Redis emulators). No Docker required.
- **Deployed:** Encore Cloud provisions the same resources in your AWS or GCP account, or you build Docker images and self-host.

## Installation

```bash
npx skills add encoredev/skills-beta
```

```bash
# List available skills
npx skills add encoredev/skills-beta --list

# Install specific skills
npx skills add encoredev/skills-beta --skill encore-beta-getting-started --skill encore-beta-database
```

### Claude Code marketplace

```bash
claude plugin marketplace add encoredev/skills-beta
claude plugin install encore-skills-beta@encore-skills-beta
```

### Manual

Copy the directories under `encore/` to your agent's skills directory. Keep each skill's `references/` folder with its `SKILL.md`.

## How the skills are organized

One skill per topic, not per language. Each topic's `SKILL.md` holds what's the same in every language (rules, CLI, gotchas), and `references/<language>.md` holds the API for TypeScript, Python, Go and Rust. An agent reads only the reference for the language it's working in, and mixed-language apps are covered by one skill.

```
encore/database/
├── SKILL.md
└── references/
    ├── typescript.md
    ├── python.md
    ├── go.md
    └── rust.md
```

## Available skills

### Setup and workflow

| Skill | Description |
|-------|-------------|
| `encore-beta-getting-started` | From nothing to a running v2 app, with pointers to each step |
| `encore-beta-install` | Install the `encore-beta` CLI and the language toolchains, match SDK versions |
| `encore-beta-login` | Log in to Encore Cloud, including with an auth key and in CI |
| `encore-beta-create-app` | `encore app new`/`init`, services from templates without prompts, `llm-rules` |
| `encore-beta-service` | `encore.service.json`, routing and exposure, listening, startup and shutdown, service calls, gateway CORS, deploy build options |
| `encore-beta-api` | Encore's TypeScript API framework (`@encore.dev/api`): `api()` endpoints, validation, middleware and auth, as in v1 |
| `encore-beta-run` | `encore check`, running in the background, status, logs, traces, fixing 404s and 502s |
| `encore-beta-mcp` | Connect AI tools to the daemon's local MCP server |
| `encore-beta-deploy` | Link to Encore Cloud, deploy, stream logs, build Docker images |
| `encore-beta-migrate-from-v1` | Move an Encore.ts or Encore.go v1 app to v2 |
| `encore-beta-code-review` | Checklist for v2 code in any language |

### Building (TypeScript, Python, Go, Rust)

| Skill | Description |
|-------|-------------|
| `encore-beta-database` | Postgres, migrations, queries, transactions, handles, ORMs |
| `encore-beta-pubsub` | Topics, subscriptions, ordered topics, retries, dead letters |
| `encore-beta-bucket` | Object storage, signed URLs, public buckets |
| `encore-beta-redis` | Redis databases for caching, counters, sessions and leaderboards |
| `encore-beta-cron` | Scheduled jobs |
| `encore-beta-secrets` | API keys and credentials |
| `encore-beta-config` | Typed, per-environment, live-reloading configuration |
| `encore-beta-observability` | Logging, metrics, OpenTelemetry tracing |
| `encore-beta-testing` | `encore test` with isolated infrastructure |

## Sources and verification

The skills are written against the v2 beta CLI and SDKs (`2.0.0-beta.5`) and the beta docs for TypeScript and Python. Where the beta docs and the CLI disagree, the skills follow the CLI and SDK source. The Go and Rust references are written from the SDK source, since those languages have no beta docs yet.

## License

Apache-2.0
