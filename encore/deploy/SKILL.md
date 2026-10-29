---
name: encore-beta-deploy
description: Deploy an Encore v2 (beta) app. Covers linking to Encore Cloud with `encore platform link` (push-to-deploy or GitHub), deploying with `git push encore` or `encore platform deploy`, streaming deployed logs, secrets for cloud environments, and self-hosting with `encore build docker`.
when_to_use: >-
  User wants to deploy an Encore v2 app, connect it to Encore Cloud, push to staging or production, deploy a branch or commit, stream production logs, or build Docker images to self-host. Trigger phrases: "deploy", "encore platform link", "link app", "git push encore", "encore platform deploy", "production", "staging", "deployed logs", "encore build docker", "self-host", "Docker image". SKIP for v1 apps (`encore app link`).
---

# Deploying an Encore v2 app

Commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

**Deploying sends code to Encore Cloud and changes live environments. Confirm with the user before linking an app, pushing to the `encore` remote, or running `encore platform deploy`, and name the target environment.**

## Before deploying

- Logged in (`encore auth whoami`, see `encore-beta-login`).
- `encore check` passes locally.
- Every exposed route is meant to be public, with auth where needed (`"expose": true` on `"prefix": "/"` publishes everything, cron routes included).
- Secrets have cloud values. During the beta, **Encore Cloud stores app-wide secrets only**: secrets the deployed app needs must be `sharedSecret`/`globalSecret` (or the language equivalent). Check with `encore secret list --missing`. Tests run during a cloud build read the `--type local` value, so give secrets the tests need one too.

**Known issue: Encore Cloud can't build TypeScript services.** The build doesn't install packages. An API framework service fails within seconds with `no node_modules directory at or above /workspace/<service>: install the service's packages`; other TS services bundle with `warning: unresolved dependency "@encore.dev/sdk"`. There's no workaround in the app: tell the user before linking, and self-host with `encore build docker`, which installs in the container, if they need it deployed now.

The cloud build also runs `encore test`, and fails with `no selected services have a "test" command` when no service has one (TS: a `test` script in `package.json`, or `"test"` in `encore.service.json`).

## 1. Link the app

```bash
encore platform link                       # creates an Encore Cloud app, records its id in encore.app.json, adds the `encore` git remote
encore platform link --name shop           # name for the new app (one-time; a suffix is added to make the slug)
encore platform link --attach shop-a1b2    # link to an existing app
encore platform link --github              # deploy through the GitHub integration instead; uses origin's repo
encore platform link --github acme/shop
```

`--new` replaces an already-recorded id (this orphans local databases named after the old id). `--force` with `--attach` links to a different app. Commit the `encore.app.json` change.

## 2. Deploy

Push-to-deploy:

```bash
git push encore main
```

GitHub-linked apps deploy from pushes to the connected repository.

Deploy a branch or commit that Encore Cloud already has, to a named environment:

```bash
encore platform deploy --env staging --branch main
encore platform deploy --env production --commit <40-char sha> --format json
```

`--env` is required. It returns as soon as the rollout starts; nothing is built locally.

## 3. Watch it

```bash
encore platform logs --env staging        # streams until interrupted; defaults to the primary environment
```

Traces and metrics for deployed environments are in the Encore Cloud dashboard.

## Self-hosting

```bash
encore build docker --tag 'registry.example.com/shop/{name}:v1'          # one image per service
encore build docker --tag '...' --push --platform linux/amd64
encore build docker --services 'users | billing' --tag '...'
encore build dist                                                        # folders on disk
encore build oci                                                         # OCI image layouts
```

`--native` assembles images without Docker. `encore.build.toml` groups services into shared images. Infrastructure for self-hosted environments is configured with `encore infra` (see `encore infra --help`).
