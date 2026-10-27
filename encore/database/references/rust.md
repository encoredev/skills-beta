# Postgres in Rust

Module: `encore::storage::postgres`, from the `encore-dev` crate.

## Declare

```rust
use encore::storage::postgres::Database;

static DB: Database = Database::new("users").migrations("./migrations");
```

- A module-level `static` with a literal name. The migrations path is relative to the file that declares the database (`src/main.rs` → `src/migrations/`).
- Another service's database: `static USERS: Database = Database::named("users");` (no options).

## Query

The query methods are `tokio_postgres::Client`'s, with `$1`, `$2`, ... parameters:

```rust
let row = DB.query_opt("SELECT id, name FROM users WHERE email = $1", &[&email]).await?; // Option<Row>
let row = DB.query_one("SELECT count(*) FROM users", &[]).await?;                      // Row
let rows = DB.query("SELECT id, name FROM users ORDER BY name", &[]).await?;            // Vec<Row>
let n: u64 = DB.execute("DELETE FROM users WHERE id = $1", &[&id]).await?;             // rows affected
DB.batch_execute("SET statement_timeout = '5s'").await?;                                // no parameters

let name: String = row.get("name");          // panics on a missing column or wrong type
let id: i64 = row.try_get("id")?;            // encore::Error instead
```

- `query_one` with no row fails with `ErrCode::NotFound`; `query_opt` and `query_one` with more than one row fail with `ErrCode::FailedPrecondition`.
- Every other database error (constraint violations included) is `ErrCode::Unavailable`, with the Postgres message in `err.message`.
- Use `INSERT ... RETURNING ...` with `query_one` to read generated values.
- Streaming: `query_raw` returns a `RowStream`.
- Types are tokio-postgres's. For `uuid`, `chrono`, `serde_json` and so on, enable the matching feature on the SDK's tokio-postgres fork:

```toml
encore-tokio-postgres = { version = "=2.0.0-beta.5", features = ["with-uuid-1", "with-chrono-0_4"] }
```

## Transactions and connections

```rust
let tx = DB.transaction().await?;
tx.execute("UPDATE accounts SET balance = balance - $2 WHERE id = $1", &[&from, &amount]).await?;
tx.execute("UPDATE accounts SET balance = balance + $2 WHERE id = $1", &[&to, &amount]).await?;
tx.commit().await?; // dropping without commit ROLLS BACK

use encore::storage::postgres::IsolationLevel;
let tx = DB.build_transaction().isolation_level(IsolationLevel::Serializable).start().await?;
```

One connection for several statements, prepared statements or `COPY`:

```rust
let conn = DB.acquire().await?;           // back to the pool on drop
let stmt = conn.prepare("SELECT name FROM users WHERE id = $1").await?;
let row = conn.query_one(&stmt, &[&id]).await?;
```

## Passing the database to other code

A `Database` can't be passed as a value (the parser rejects it). Pass a handle; library code takes `&impl Queryable`:

```rust
use encore::storage::postgres::{DatabaseRef, Queryable};

async fn count(db: &impl Queryable) -> Result<i64, encore::Error> {
    Ok(db.query_one("SELECT count(*) FROM users", &[]).await?.get(0))
}

count(&DB.read()).await?;
static REPORTS: DatabaseRef = DB.read(); // handles are const
```

| Handle | Access |
|---|---|
| `DB.read()` | Read-only; may use a read replica. Writes fail with SQLSTATE `25006`. |
| `DB.read_primary()` | Read-only, always the primary |
| `DB.write()` | Read and write |
| `DB.admin()` | Read, write and administration |

Query methods on the `static` itself count as write access. `DatabaseRef`, `Transaction` and `Connection` implement `Queryable`; `Database` does not.

## ORMs and other clients

```rust
let url = DB.connection_string()?;      // read-write, through the local proxy
let ro = DB.read().connection_string()?; // read-only session
```

Use it with sqlx, Diesel or SeaORM. Queries through another client are not in Encore's database spans.
