# Testing in Rust

`encore test` runs each Rust service's default `test` strategy: `cargo test --features=encore-dev/local` in the service directory. Ordinary `#[test]`, `#[tokio::test]` and `#[encore::test]` tests all work.

```bash
encore test                                   # every service
encore test --services api                    # one service
encore test -- cargo test --features encore-dev/local creates_a_user   # a command of your own
```

Keep `--features encore-dev/local` in your own command, or cargo rebuilds the SDK with different features. Or set it in `encore.service.json`: `"test": { "type": "custom", "cmd": [...] }`.

Run tests through `encore test`: a plain `cargo test` has no runtime config, and the SDK exits the process on the first resource use when it can't initialize.

## Isolation

Every test gets its own infrastructure: a freshly migrated, empty database, empty buckets and Redis databases, and its own topics. Tests can run in parallel without cleanup. The test is worked out from the thread: the test's own, or a thread of the Tokio runtime it runs in.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[encore::test] // #[tokio::test] that registers the test up front; takes flavor/worker_threads
    async fn creates_a_user() -> Result<(), encore::Error> {
        create_user("ada").await?;
        let row = DB.query_one("SELECT count(*) FROM users", &[]).await?;
        assert_eq!(row.get::<_, i64>(0), 1);
        Ok(())
    }
}
```

A thread you spawn yourself must join the test: `let scope = encore::testing::current()?;` then `let _guard = scope.enter();` on that thread.

## The `testing` module

```rust
encore::testing::set_secret(&STRIPE_KEY, "sk_test")?;                          // this test only
encore::testing::set_config(&CHECKOUT, serde_json::json!({ "max_items": 2, "currencies": [], "payment_timeout": "1s" }))?; // checked against the schema
```

Config values otherwise come from the files for the `test` environment type.

## Pub/Sub

No subscription runs under `encore test`. Read what the test published and call the handler yourself:

```rust
place_order("o1").await?;
let published = encore::testing::published(&ORDER_PLACED)?; // Vec<Message<OrderPlaced>>
assert_eq!(published.len(), 1);
send_confirmation(published[0].clone()).await?;
```

## Fixtures

Start tests from a saved state instead of an empty one. The setup runs once per run; each test gets a copy:

```rust
encore::fixture! {
    /// Two users.
    pub static SEEDED = async {
        DB.execute("INSERT INTO users (name) VALUES ('ada'), ('grace')", &[]).await?;
        Ok(())
    };
}

#[encore::test(fixture = SEEDED)] // or call SEEDED.load().await? before any resource use
async fn lists_users() { /* ... */ }
```

`pub static SEEDED for mod = ...;` makes it the default for every `#[encore::test]` in that module; opt out with `#[encore::test(fixture = none)]`. Databases, buckets and Redis are saved; topics always start empty.

## Other services

`services::base_url("orders")` returns a URL, but requests get `501 Not Implemented` (a generated client returns `Error::Status { status: 501, .. }`, its body naming the service and path called). Mock the call, for example by giving the client a base URL of a local stub server.

## Cron

Not scheduled. Call the handler, or the function behind it, directly.
