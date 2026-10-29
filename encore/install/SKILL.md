---
name: encore-beta-install
description: Install, update and find the Encore v2 beta CLI (`encore-beta`), keep it next to a v1 install, and install the language toolchains Encore runs services with (Bun or Node.js, uv, Go, Cargo). Also covers matching the SDK version to the CLI.
when_to_use: >-
  User wants to try or set up Encore v2 / the Encore beta, install or update the beta CLI, or sees `encore-beta: command not found`, a v1 `encore` answering when v2 was intended, an SDK/CLI version mismatch, or a service failing with "failed to spawn bun/uv/go/cargo". Trigger phrases: "install Encore beta", "encore-beta", "Encore v2", "brew install encore-beta", "update the beta", "command not found", "version mismatch", "failed to spawn". SKIP for the stable v1 CLI.
---

# Installing the Encore v2 beta CLI

## The CLI name

The beta installs as **`encore-beta`**, next to any v1 `encore`, so v1 apps keep working. The Encore v2 docs and these skills write commands as `encore ...`. **When you run them, use `encore-beta` unless `encore version` reports `2.x`.** Shell aliases don't apply in an agent's non-interactive shell, so don't rely on `alias encore=encore-beta`.

```bash
command -v encore-beta && encore-beta version     # e.g. 2.0.0-beta.5+<sha>
command -v encore && encore version               # v1 prints 1.x
```

## Install

macOS and Linux, through Encore's Homebrew tap:

```bash
brew install encoredev/tap/encore-beta
encore-beta version
```

Update:

```bash
brew upgrade encoredev/tap/encore-beta
encore-beta version upgrade        # 2.0.0-beta.6+: upgrades in place, to the latest on its channel
```

**Restart the daemon after upgrading.** The old daemon keeps running from the removed version's directory, and `encore version` prints the **daemon's** version, so it still reports the old one. Until it restarts, `encore test` fails with `could not determine the Encore installation root, so the bundled Postgres could not be found` (setting `ENCORE_POSTGRES_ROOT` in your shell doesn't help: the daemon provisions databases), and under `encore run` or `encore check` every TS service fails with `could not find the Encore runtime library ('encore-runtime.node')`, so the gateway answers 502. There's no restart command (2.0.0-beta.7). Compare `ls -l "$(command -v encore-beta)"` with the binary of the running `encore-beta daemon` process, and if they differ, stop that process (`pkill -f "encore-beta daemon"`, which stops the apps it runs) and start it again with `encore-beta daemon`.

**v1 and v2 daemons both want port 9400** for the dev dashboard. With a v1 `encore` daemon running, the v2 daemon can retry, then exit without an error (`encore-beta daemon` still prints that it's running; check its log at `~/Library/Caches/encore/daemon.log` on macOS). Start it on another port: `ENCORE_DEVDASH_LISTEN_ADDR=127.0.0.1:9401 encore-beta daemon`.

There's no Windows build documented for the beta; use WSL with Homebrew on Linux.

The CLI starts a background daemon on first use. Local infrastructure needs nothing else: Postgres runs from a bundled native build by default, and Pub/Sub, object storage and Redis are emulated. **Docker is not required** to run apps; it's only used by `encore build docker`.

## Language toolchains

Encore starts each service with its own toolchain, so install what the services use:

| Language | Needs | Notes |
|---|---|---|
| TypeScript | Bun or Node.js, plus the package manager matching the lockfile | The official templates run on Bun (`brew install oven-sh/bun/bun`). |
| Python | The package manager matching the lockfile: usually `uv` (`brew install uv`), or poetry, pipenv, pdm | Templates use `uv` and Python 3.14. |
| Go | Go matching the app's `go.mod` | |
| Rust | `cargo` (rustup), a recent stable | The SDK doesn't compile on Rust 1.89; 1.99 works. `rustup update stable`. |

A missing toolchain shows up when the service starts: `failed to spawn uv for unit notes: No such file or directory`, and requests to it get `502`.

## SDK version must match the CLI

Each SDK loads the native runtime that ships with the CLI, so **pin the SDK to the CLI's version**. With CLI `2.0.0-beta.5`:

| Language | Dependency |
|---|---|
| TypeScript | `@encore.dev/sdk@2.0.0-beta.5` |
| Python | `encore-dev==2.0.0b5` |
| Go | module `sdk.encore.dev` (see the Go reference in `encore-beta-service`) |
| Rust | crate `encore-dev` (see the Rust reference in `encore-beta-service`) |

`encore service new` pins this for you. After upgrading the CLI, bump every service's SDK pin to the new version.

## Next

Log in (`encore-beta-login`), or go straight to creating an app (`encore-beta-create-app`). Logging in is only needed for Encore Cloud.
