---
name: encore-beta-secrets
description: Manage API keys, credentials and signing keys in an Encore v2 (beta) app in TypeScript, Python, Go or Rust. Covers service, shared and global secret scopes, reading values, `encore secret set` locally and per Encore Cloud environment, `.secrets.user.toml`, and listing missing secrets.
when_to_use: >-
  User is in an Encore v2 app and needs a third-party API key, database password, webhook signing secret, OAuth client secret or JWT key without committing it, or sees an error that a secret has no value. Trigger phrases: "secret", "API key", "credentials", "Stripe key", "OpenAI key", "encore secret set", ".secrets.user.toml", ".env", "secret not set", "KeyError secret". SKIP for non-sensitive settings (`encore-beta-config`) and v1 apps (`.secrets.local.cue`).
---

# Secrets in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

Declare a secret by name in code; set its value outside the code, per machine or per Encore Cloud environment.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`).

## Scopes

| Scope | TS / Python | Meaning |
|---|---|---|
| Service | `secret("X")` / `Secret[str]("X")` | The declaring service's own. Two services can each have `X` with different values. Must be declared inside a service. |
| Shared | `sharedSecret("X")` / `SharedSecret[str]("X")` | One app-wide value; Encore tracks which services read it. Can be declared outside a service. |
| Global | `globalSecret("X")` / `GlobalSecret[str]("X")` | One app-wide value, available to every service, untracked. |

Start with a service secret. **Exception for deployed apps: during the beta, Encore Cloud stores app-wide secrets only.** A service's own secret can only get a value locally, so anything that must work when deployed needs a shared or global secret.

## Setting values locally

```bash
encore secret set payments.StripeKey          # a service's own secret: <service>.<Name>
encore secret set --service payments StripeKey
encore secret set SigningKey                  # shared/global: the bare name
cat key.pem | encore secret set SigningKey    # stdin works; otherwise it prompts without echo
encore secret set                             # prompt for every declared secret with no value
```

Values go in `.secrets.user.toml` at the app root (gitignored by `encore app new`). **Never commit it.** You can edit it directly:

```toml
[services.payments]
StripeKey = "sk_test_..."

[shared]          # shared and global secrets
SigningKey = "..."
```

Setting a value non-interactively from an agent: pipe it on stdin (`printf %s "$VALUE" | encore secret set X`), never on the command line. Ask the user for real credentials rather than inventing them.

## Encore Cloud environments

```bash
encore secret set --type production SigningKey
encore secret set --type development,preview,local SigningKey   # types: production (prod), development (dev), preview (pr), local
encore secret set --env staging SigningKey                     # a named environment; overrides its type's value
```

`--type local` stores a value on Encore Cloud that every developer's `encore run` and `encore test` receive; `.secrets.user.toml` overrides it on your machine.

## Checking

```bash
encore secret list              # where each secret has values; never prints them
encore secret list --missing    # only declared secrets without a value; exits non-zero if any (good in CI)
encore secret unset SigningKey  # takes the same target flags as set
```

## Missing values

- TypeScript: a missing secret reads as `""` locally, and throws when deployed.
- Python: reading `.value` raises `KeyError`, locally and deployed.
- Check the Go and Rust references for their behavior.
