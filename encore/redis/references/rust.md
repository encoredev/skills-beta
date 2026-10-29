# Redis in Rust

Module: `encore::storage::redis`. Commands follow the `redis` crate's names and conversion traits.

## Declare

```rust
use encore::storage::redis::{Database, EvictionPolicy};

static SESSIONS: Database = Database::new("sessions");                 // a cache: may evict
static LEDGER: Database = Database::new("ledger").persistent();        // never evicted
static PAGE_CACHE: Database =
    Database::new("page-cache").eviction_policy(EvictionPolicy::AllKeysLfu); // default AllKeysLru
```

Name: kebab-case literal. `persistent()` takes no eviction policy. Another service's database: `Database::named("sessions")`.

## Commands

Grouped by data type, named after the Redis command (snake_case):

```rust
use encore::storage::redis::{ExistenceCheck, SetExpiry, SetOptions};

SESSIONS.string().set(format!("session:{id}"), &token).await?;
SESSIONS.string().set_ex(format!("otp:{user_id}"), code, 300).await?;      // seconds
SESSIONS.string().set_nx("lock:job", &token).await?;                        // bool
let _: Option<String> = SESSIONS.string().set_options(key, value,
    SetOptions::default().conditional_set(ExistenceCheck::NX).with_expiration(SetExpiry::EX(60))).await?;
let visits: i64 = SESSIONS.string().incr(format!("visits:{user_id}"), 1).await?; // f64 delta → f64
SESSIONS.hash().hset(format!("user:{user_id}"), "name", &name).await?;
SESSIONS.list().lpush("jobs", &job_id).await?;
SESSIONS.set().sadd(format!("tags:{post_id}"), "rust").await?;
SESSIONS.sorted_set().zadd("leaderboard", &user_id, score).await?;         // member, then score
SESSIONS.generic().expire(format!("session:{id}"), 3600).await?;
SESSIONS.generic().del(format!("session:{id}")).await?;
```

Groups: `string`, `hash`, `list`, `set`, `sorted_set`, `stream`, `geo`, `bitmap`, `pubsub`, `generic`.

**Call a command in the same expression as its group** (`SESSIONS.string().get(..)`). Storing or passing a group is a parse error.

## Reading replies

Fixed replies have fixed types (`del` → `u64`, `expire` → `bool`). Value replies take the type you annotate:

```rust
let name: Option<String> = SESSIONS.string().get(format!("name:{user_id}")).await?; // None if missing
let user: HashMap<String, String> = SESSIONS.hash().hgetall(format!("user:{user_id}")).await?;
let top: Vec<(String, f64)> = SESSIONS.sorted_set().zrange_withscores("leaderboard", 0, 9).await?;

use encore::storage::redis::Json;
SESSIONS.string().set("user:1", Json(&user)).await?;
let Json(user): Json<Option<User>> = SESSIONS.string().get("user:1").await?; // missing = None
```

Read possibly missing keys as `Option<T>` (a missing key read as plain `String` is an error). Errors are `encore::Error`: `ErrCode::Unavailable` for the server or connection, `ErrCode::Internal` when the reply isn't the type you asked for.

## Pipelines, transactions, watch, scripts

```rust
let mut pipe = SESSIONS.pipeline();          // one round trip, not atomic
pipe.string().get("name:1");
pipe.string().get("name:2");
let (a, b): (Option<String>, Option<String>) = pipe.exec().await?;

let mut tx = SESSIONS.transaction();         // MULTI/EXEC, nothing sent until exec
tx.string().incr("hits", 1);
let (hits,): (i64,) = tx.exec().await?;

let left: i64 = LEDGER.watch("balance", async |conn, tx| { // optimistic, retried on conflict
    let balance: i64 = conn.string().get("balance").await?;
    tx.string().set("balance", balance - 10);
    Ok(balance - 10)
}).await?;

use encore::storage::redis::Script;
static RELEASE: Script = Script::new(r#"if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) end return 0"#);
let ok: bool = SESSIONS.script(&RELEASE).key(format!("lock:{job}")).arg(&token).invoke().await?;
```

`watch` gives up with `ErrCode::Aborted` after 64 conflicting attempts. Pass every key a script touches with `.key()`: only those get the database's key prefix. `Script::read_only(..)` runs on a read handle.

## Passing the database to other code

```rust
use encore::storage::redis::ReadOnlyDatabaseRef;

async fn name_of(db: ReadOnlyDatabaseRef, user: &str) -> Result<Option<String>, encore::Error> {
    db.string().get(format!("name:{user}")).await
}
name_of(SESSIONS.read(), &user_id).await?;
```

Handles (`Copy`, const): `read()` (may use a replica), `read_primary()`, `write()` (`DatabaseRef`). A read handle only has read commands.

## Other clients

`SESSIONS.connection_string()?` works with other Redis clients, but bypasses the key prefix, so keys don't match the SDK's.
