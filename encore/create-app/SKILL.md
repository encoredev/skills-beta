---
name: encore-beta-create-app
description: Create a new Encore v2 (beta) app with `encore app new`, turn an existing repository into one with `encore app init`, add services from templates with `encore service new` without interactive prompts, and install agent instruction files with `encore llm-rules init`.
when_to_use: >-
  User wants a new Encore v2 app or project, wants to adopt an existing repo or backend into Encore v2, wants to scaffold a service from a template (FastAPI, Django, Flask, Fastify, Next.js, Prisma, Axum, hello world, Stripe, Clerk, Resend, Slack bot, uptime monitor, URL shortener), or a create command hangs on a picker. Trigger phrases: "new Encore v2 app", "encore app new", "encore app init", "encore service new", "template", "scaffold", "adopt this repo", "llm-rules". SKIP for v1's `encore app create`.
---

# Creating an Encore v2 app

Commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

An app is a directory with `encore.app.json`. Services are added separately, each a directory with `encore.service.json`.

## New app

```bash
encore app new my-app --llm-rules none     # or agentsmd, cursor, claudecode, vscode, zed (repeatable)
cd my-app
```

- Creates `my-app/` (must not exist or must be empty), `encore.app.json` (`{"$schema": ...}`), a `.gitignore` covering `.encore`, `.secrets.user.toml` and `*.user.config.*`, and a git repo with a first commit.
- Inside an existing repository it adds the app as a subfolder.
- The name is a single directory name, not a path. **There's no `--template` on `app new`**; templates are per service.
- Pass `--llm-rules` explicitly: without it, it uses the `bootstrap.llm-rules` setting.

## Existing repository

```bash
encore app init                 # in the repo root; writes encore.app.json, leaves code alone
encore app init path/to/dir
```

Then add an `encore.service.json` to each service directory (see `encore-beta-service`), or migrate infrastructure with `encore migrate` (see `encore-beta-migrate-from-v1` for v1 apps).

## Add services from templates

The interactive picker blocks an agent. **Always pass `--ui plain` (or `json`) and `--template`:**

```bash
encore service new --ui plain                                   # list templates
encore service new --ui json --lang python                      # filter: --lang, --category integration|framework|demo
encore service new --ui plain --describe "stripe payments"      # search
encore service new api --template typescript-fastify --ui plain
encore service new notes --template python-fastapi --ui plain --dir services/notes
```

The name is snake_case and also the directory unless `--dir` says otherwise. It writes the files, pins the SDK to the CLI version, and installs dependencies (`--no-install` prints the commands instead).

Templates in 2.0.0-beta.5 (list them for the current set):

| Category | Templates |
|---|---|
| Frameworks | `typescript-fastify`, `typescript-nextjs`, `typescript-prisma`, `python-fastapi`, `python-django`, `python-flask`, `rust-axum` |
| Integrations | `typescript-clerk`, `typescript-sentry`, `typescript-stripe`, `typescript-resend`, `typescript-slack-bot`, `go-slack-bot` |
| Demos | `{typescript,python,go,rust}-hello-world`, `{typescript,go}-todo-api`, `-url-shortener`, `-uptime-monitor`, `-webhook-processor`, `typescript-file-upload`, `typescript-signup-events` |

**Template services are not exposed on the gateway by default.** Their `api` entry has no `"expose": true`, so requests through the gateway get `404`. Add `"expose": true` to reach them from outside (and protect them before deploying).

## Agent instruction files

```bash
encore llm-rules init --llm-rules claudecode --lang typescript   # repeatable flags; --force to replace
```

Writes Encore's own instructions for the chosen tools (`agentsmd`, `cursor`, `claudecode`, `vscode`, `zed`), for the languages it detects or the ones you pass.

## Next

Run and verify it with `encore-beta-run`.
