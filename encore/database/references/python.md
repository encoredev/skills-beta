# Postgres in Python

Import: `from encore.storage import postgres` (package `encore-dev`).

## Declare

```python
# orders/db.py
from encore.storage import postgres

db = postgres.Database("orders", migrations="./migrations")
```

- `migrations` is keyword-only: a string literal relative to the declaring file (no absolute paths, no `..`), and the directory must exist.
- Leave it out when something else owns the schema (Django migrations, Alembic).

## Query

Parameters are positional, referenced as `$1`, `$2`, ...

```python
row = await db.query(
    "INSERT INTO orders (customer, total_cents) VALUES ($1, $2) RETURNING id",
    customer, total_cents,
).one()                                   # exactly one row; raises EncoreError on 0 or >1 rows

order = await db.query("SELECT * FROM orders WHERE id = $1", order_id).first()   # dict or None

rows = await db.query("SELECT id FROM orders WHERE customer = $1", customer).all()

async for row in db.query("SELECT total_cents FROM orders WHERE customer = $1", customer):
    total += row["total_cents"]

result = await db.exec("UPDATE orders SET status = 'shipped' WHERE id = $1", order_id)
result.rows_affected
```

- `query` returns a `Cursor` that runs nothing until consumed, and can be consumed **once** (`one()`, `first()`, `all()`, or `async for`). A second use raises `EncoreError`.
- **`one()` raises when no row matches.** For a lookup that may miss, use `first()`. (The beta docs say `one()` returns `None`; it doesn't.)
- Stopping iteration early: `async with db.query(...) as rows:` so it closes.
- Rows are dicts keyed by column name.

### Types

Rows use the types asyncpg and psycopg use, and parameters accept the same types:

| Postgres | Python |
|---|---|
| `numeric` | `decimal.Decimal` (keeps the column's scale) |
| `timestamptz` | aware `datetime` in UTC (a naive `datetime` parameter is taken as UTC) |
| `timestamp` | naive `datetime` |
| `date`, `time` | `datetime.date`, `datetime.time` |
| `bytea` | `bytes` |
| `json`, `jsonb` | decoded JSON value (a `dict`/`list` parameter is stored as JSON) |
| arrays | `list` |
| `uuid`, `inet`, `cidr` | `str` |

The beta docs' type table (numeric and timestamps as strings, no `datetime` parameters) is out of date.

## Transactions and connections

```python
async with db.begin() as tx:              # commits on normal exit, rolls back on exception
    await tx.exec("INSERT INTO orders (customer, total_cents) VALUES ($1, $2)", customer, total)

async with db.acquire() as conn:          # several statements on one connection
    await conn.exec("SELECT pg_advisory_lock($1)", order_id)
```

Unlike TypeScript, the context manager commits on success. `tx = await db.begin()` with explicit `commit()`/`rollback()` also works. Close open cursors before committing.

## Passing the database to other code

```python
from encore.storage import postgres

async def find_order(conn: postgres.Queryable, order_id: int):
    return await conn.query("SELECT * FROM orders WHERE id = $1", order_id).one()

await find_order(db.read(), order_id)    # db.read(), db.write(), db.admin()
```

FastAPI dependency:

```python
async def sql_read() -> postgres.DatabaseRef:
    return db.read()
SqlReader = Annotated[postgres.DatabaseRef, Depends(sql_read)]
```

A `read()` handle runs on read-only connections: a write fails with `EncoreError` (SQLSTATE 25006), and its transactions are read-only. It reads from a replica when the environment has one; `read(primary=True)` reads from the primary. `write()` and `admin()` handles are read-write.

Another service's database: `orders_db = postgres.Database.named("orders")` at module scope.

## ORMs and other clients

`db.connection_string` points at an in-process proxy, valid only in a process running the SDK. Those queries aren't traced. A `read()` handle's `connection_string` opens read-only sessions.

Django (what the `python-django` template does):

```python
# notes/resources.py
db = postgres.Database("dj")              # no migrations: Django owns the schema

# config/settings.py
from psycopg.conninfo import conninfo_to_dict
from notes.resources import db

c = conninfo_to_dict(db.admin().connection_string)   # admin: Django's migrations run DDL
DATABASES = {"default": {
    "ENGINE": "django.db.backends.postgresql",
    "NAME": c.pop("dbname"), "USER": c.pop("user", ""), "PASSWORD": c.pop("password", ""),
    "HOST": c.pop("host", ""), "PORT": c.pop("port", ""), "OPTIONS": c,
}}
```

Encore doesn't run Django's migrations. The template runs `call_command("migrate", interactive=False)` in `main.py` before `encore.run(application)` (the ASGI app). Other management commands go through `encore exec`:

```bash
encore exec -- uv run python manage.py makemigrations
```

SQLAlchemy and others: pass the connection string, and manage migrations with their own tools.

## Errors

Database errors raise `encore.EncoreError` (a `RuntimeError`). It has no Postgres error code yet; only the message tells a unique violation from a syntax error.
