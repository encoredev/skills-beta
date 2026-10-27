# Cron jobs in Rust

Module: `encore::cron`.

```rust
use encore::cron::{Job, Schedule};

static CLEANUP: Job = Job::new(
    "cleanup-expired-sessions",
    "/internal/cleanup",
    Schedule::Every("1h"),          // or Schedule::Cron("0 4 * * *") (UTC)
)
.title("Delete expired sessions"); // optional
```

- Name: lowercase letters, digits and dashes. Path: a literal starting with `/`.
- `Every` must divide 24 hours evenly (`"30m"`, `"6h"`; not `"50m"` or `"7h"`), at most `"24h"`.

Serve the path as a `POST` route, using the job's own path so they stay in step:

```rust
async fn cleanup() -> Result<String, StatusCode> {
    let deleted = DB.execute("DELETE FROM sessions WHERE expires_at < now()", &[]).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(format!("deleted {deleted}"))
}

let app = Router::new().route(CLEANUP.path(), post(cleanup));
```

The service gets the full path, so with an `/api` prefix the job's path is `/api/internal/cleanup`.
