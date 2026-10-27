---
name: encore-beta-config
description: Typed, live-reloading configuration in an Encore v2 (beta) app, in TypeScript, Python, Go or Rust. Covers declaring a config type, TOML/JSON value files beside the service, per-environment overlays, user overrides, constraints, secret fields, and the `encore config` commands.
when_to_use: >-
  User is in an Encore v2 app and wants feature flags, limits, timeouts, allowed values or other non-secret settings that differ per environment, or wants to change a setting without a deploy. Trigger phrases: "config", "configuration", "feature flag", "setting", "per environment", "production value", "checkout.config.toml", "encore config set", "environment variable for settings". SKIP for credentials (`encore-beta-secrets`).
---

# Config in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

A config is a typed value. You declare its type in code and write its value in files beside the code. Encore validates the files against the type before your code sees the value, and pushes changes to running services without a restart.

Prefer config over branching on the environment name in code.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`).

## Files

For a config named `checkout`, beside the declaring service's `encore.service.json` (or beside `encore.app.json` for shared/global configs):

| File | Applies to |
|---|---|
| `checkout.config.toml` | Every environment |
| `checkout.<type>.config.toml` | Environment type: `local`, `test`, `development`, `preview`, `production` |
| `checkout.@prod-eu.config.toml` | One named environment |
| `checkout.user.config.toml` | Your own `encore run` only. Gitignored, never used in tests |

`.json` works instead of `.toml`. More specific layers override less specific ones field by field (JSON Merge Patch). Each file is checked as a partial value, then the merged value in full. A file whose name is close to a config's is reported as a likely typo.

Durations are written as strings: `"30s"`, `"250ms"`, `"1h30m"`.

```toml
# shop/checkout.config.toml
maxItems = 100
currencies = ["USD", "EUR"]
paymentTimeout = "30s"
```

## Scopes

Like secrets: service config (the declaring service's own), shared (one value, readers tracked), global (one value, every service). Configs never hold secret values. A field typed as a secret reads the secret of that name instead; config files can't set it.

## CLI

```bash
encore config list                                   # every config, and whether its value is valid
encore config show shop.checkout                     # merged value, and which file each field came from
encore config show shop.checkout --env-type production
encore config validate                               # every file, for every environment type
encore config schema shop.checkout                   # the JSON Schema derived from the type
encore config set shop.checkout maxItems 200         # writes the user override by default
encore config set shop.checkout maxItems 500 --env-type production   # or --base, --env <name>
encore config unset shop.checkout maxItems
encore config link                                   # add schema references for editor validation
```

Name a service's config `<service>.<name>`, a shared or global one by name. Paths look like `maxItems`, `promo.code`, `rules[0].id`. A value that parses as JSON is read as JSON; pass `--string` to force a string.

## Reading

**Read a config inside a service.** Encore grants a config to the services that read it, so a call from shared code outside any service fails with ENC0035, even for a `globalConfig`. Declare it once in shared code and have each service call it.

Read the value where you use it (it can change while running). Each language also has an on-change callback for things like resizing a pool. Tests use the base file plus the `test` overlay, never user overrides.
