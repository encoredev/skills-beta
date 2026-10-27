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
).one()                                   # dict, or None; raises EncoreError on >1 row

rows = await db.query("SELECT id FROM orders WHERE customer = $1", customer).all()

async for row in db.query("SELECT total_cents FROM orders WHERE customer = $1", customer):
    total += row["total_cents"]

result = await db.exec("UPDATE orders SET status = 'shipped' WHERE id = $1", order_id)
result.rows_affected
```

- `query` returns a `Cursor` that runs nothing until consumed, and can be consumed **once** (`one()`, `all()`, or `async for`).
- Stopping iteration early: `async with db.query(...) as rows:` so it closes.
- Rows are dicts keyed by column name.

### Types

Parameters: `None`, `bool`, `int`, `float`, `str`, `bytes`, `bytearray`, `list`, `tuple`, `dict`. **`datetime` and `Decimal` aren't supported yet**: pass strings (`"2026-01-31T12:00:00Z"`).

Results: `numeric` → `str`, `timestamp`/`timestamptz`/`date`/`time` → `str` (RFC 3339), `uuid` → `str`, `bytea` → `bytes`, `json`/`jsonb` → Python JSON values, arrays → `list`.

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

The access level is recorded for permissions only: today a `read()` handle can still write, and `read(primary=True)` has no effect yet.

Another service's database: `orders_db = postgres.Database.named("orders")` at module scope.

## ORMs and other clients

`db.connection_string` points at an in-process proxy, valid only in a process running the SDK. Those queries aren't traced.

Django:

```python
# storage/resources.py
db = postgres.Database("django")          # no migrations: Django owns the schema

# config/settings.py
from psycopg.conninfo import conninfo_to_dict
from storage.resources import db

c = conninfo_to_dict(db.connection_string)
DATABASES = {"default": {
    "ENGINE": "django.db.backends.postgresql",
    "NAME": c.pop("dbname"), "USER": c.pop("user", ""), "PASSWORD": c.pop("password", ""),
    "HOST": c.pop("host", ""), "PORT": c.pop("port", ""), "OPTIONS": c,
}}
```

```bash
encore exec -- uv run python manage.py migrate
```

SQLAlchemy and others: pass the connection string, and manage migrations with their own tools.

## Errors

Database errors raise `encore.EncoreError` (a `RuntimeError`). It has no Postgres error code yet; only the message tells a unique violation from a syntax error.
