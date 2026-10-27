---
name: encore-beta-database
description: Use PostgreSQL in an Encore v2 (beta) app in TypeScript, Python, Go or Rust. Covers declaring a database, SQL migrations, parameterized queries, transactions, read/write handles, ORMs through the connection string, sharing a database between services, and the `encore db` commands.
when_to_use: >-
  User is in an Encore v2 app and wants a Postgres table, a migration, a query, an insert/update/delete, a transaction, an ORM (Drizzle, Prisma, Django ORM, SQLAlchemy, sqlx, GORM) on an Encore database, seed data, or a psql shell. Trigger phrases: "database", "Postgres", "table", "migration", "SQL", "queryRow", "query", "transaction", "Drizzle", "Prisma", "Django", "seed", "encore db shell", "migration failed", "dirty database". SKIP for v1 apps using `SQLDatabase` from `encore.dev/storage/sqldb`: use the stable skills.
---

# Postgres databases in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

Declare a database in code. Encore provisions it: a local Postgres cluster for `encore run`, an isolated copy per test run, RDS or Cloud SQL when deployed.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`) for the declaration, query API and transaction semantics. They differ: TypeScript rolls back an uncommitted transaction on dispose, Python's `async with` commits on success.

## Rules for every language

- Declare at module/package scope, name as a string literal, unique in the app. **Never rename a deployed database**: Encore would create a new, empty one.
- Keep the declaration in its own module (`db.ts`, `db.py`, ...) and import it, so tests and scripts can use it without starting the server.
- The database belongs to the declaring service. Another service binds it by name (`Database.named("users")` or equivalent), but prefer one owner exposing the data over its API.
- **Always parameterize.** Never build SQL from request input.
- To hand the database to a helper, pass a handle (`read()`, `write()`, `admin()`), not the declaration.

## Migrations

```
users/migrations/1_create_users.up.sql
users/migrations/2_add_status.up.sql
users/migrations/2_add_status.down.sql   # optional; Encore never runs it on its own
```

- Named `<number>_<description>.up.sql`, applied in ascending order before the service starts, recorded in `schema_migrations`.
- **Never edit an applied migration.** Add a new one with a higher number. A new file numbered below the latest applied one is never run.
- Keep migrations compatible with the previous release's code, which may still be running during a deploy.
- A migration runs as one batch. If it fails, the database is marked dirty at that version.
- ORM-generated migrations: set the migrations source (`drizzle`, `drizzle/v1` or `prisma`, see the reference). If the ORM owns the schema entirely (Django), declare the database without migrations and run the ORM's migrations through `encore exec`.

## CLI

All `encore db` commands take the database as `--database <name>`:

```bash
encore db list
encore db shell --database users                  # psql
encore db query --database users "SELECT count(*) FROM users"
encore db conn-uri --database users               # connection string for other tools
encore db proxy                                   # local proxy to every database
encore db version --database users                # version and dirty flag
encore db migrate --database users --up           # or --down, --to N, --set N
encore db reset --database users                  # drop and recreate locally
```

Recovering from a failed migration: `encore db migrate --database users --set <last good version>` (`--set=-1` if the first failed), fix the file, run again.

## Seed data

- Data every environment needs (lookup tables): a migration.
- Local dev data: a script run with `encore exec`, which attaches it to the same local databases as `encore run`:

```bash
encore exec bun scripts/seed.ts
encore exec -- uv run python seed.py
```

Tests start from the migrated schema, never from local data.
