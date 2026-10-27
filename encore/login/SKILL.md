---
name: encore-beta-login
description: Log the Encore v2 beta CLI in to Encore Cloud, in a browser or without one using an auth key, check who is logged in, and authenticate in CI with `ENCORE_AUTH_KEY`.
when_to_use: >-
  User wants to log in or out of Encore Cloud with the v2 beta CLI, check the logged-in account, authenticate a headless machine or CI, or a v2 command fails because it isn't logged in. Trigger phrases: "encore auth login", "log in to Encore", "not logged in", "auth key", "ENCORE_AUTH_KEY", "whoami", "CI login". SKIP when only running locally: no login is needed for `encore run`, `encore test` or `encore check`.
---

# Logging in to Encore Cloud (v2 beta)

Commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

Local development needs no login. Log in to link an app to Encore Cloud, deploy, or set secrets for cloud environments.

```bash
encore auth whoami          # who is logged in, if anyone
encore auth login           # opens a browser to approve
encore auth logout
```

## Without a browser

`encore auth login` waits for a browser approval, so an agent shouldn't run it unattended. Either hand it to the user (`! encore-beta auth login` in Claude Code runs it in their session), or use an auth key from the Encore Cloud dashboard:

```bash
encore auth login --auth-key                    # prompts with echo off
echo "$ENCORE_AUTH_KEY" | encore auth login --auth-key
```

Don't pass the key as an argument (`--auth-key KEY`): it lands in shell history and process listings. Never write a key into the repo.

## CI

Every command falls back to `$ENCORE_AUTH_KEY`, so CI usually needs no explicit login: set the variable from the CI provider's secret store.
