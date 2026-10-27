# Rust services

## Install the SDK

The Cargo package is `encore-dev`; the crate is `encore`. Pin it to the CLI's version:

```toml
[dependencies]
encore-dev = "=2.0.0-beta.5"   # `encore version` reports the CLI's
log = { version = "0.4", features = ["kv"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net"] }
axum = "0.8"
```

`encore service new <name> --template rust-axum` (or `rust-hello-world`) writes this for you. The SDK needs a recent stable Rust: 1.89 fails to compile it, 1.99 works.

## Run strategy

`{"type":"rust"}` is the default for `"lang": "rust"`, so the manifest can leave `run` out:

```json
{ "name": "api", "lang": "rust", "run": { "type": "rust" },
  "api": [{ "type": "http", "prefix": "/api", "name": "api", "expose": true }] }
```

It runs `cargo build`, then `cargo run`, with `--features=encore-dev/local` (the local infrastructure providers). A custom command must pass that feature itself:

```json
"run": { "type": "custom", "cmd": ["cargo", "run", "--bin", "api", "--features", "encore-dev/local"] }
```

Without `"expose": true` the gateway answers 404 for the service's routes.

## Server

Keep the framework. Initialize the SDK, listen on `ENCORE_LISTEN_ADDR` and drain on shutdown:

```rust
#[tokio::main]
async fn main() {
    encore::initialize(); // fails fast on bad runtime config; starts registered subscriptions
    encore::log::init();  // `log` records in Encore's format

    let app = axum::Router::new().route("/api/hello", axum::routing::get(|| async { "hi" }));

    let addr = std::env::var("ENCORE_LISTEN_ADDR").unwrap_or_else(|_| "127.0.0.1:8080".into());
    let listener = tokio::net::TcpListener::bind(&addr).await.expect("bind");
    axum::serve(listener, app)
        .with_graceful_shutdown(encore::shutdown::signal())
        .await
        .expect("serve");
    std::process::exit(encore::shutdown::drained().await.code());
}
```

The service gets the full path, its prefix included (`/api/hello`, not `/hello`).

On `SIGTERM`/`SIGINT`, Encore stops taking Pub/Sub messages, waits for handlers, flushes telemetry and exits (0 within the deadline, 1 after it). Awaiting `shutdown::signal()` defers that until `drained()` is called. A process that serves no HTTP calls `encore::run_blocking()` instead.

`initialize()` is optional (resources initialize on first use), but it is what starts `#[encore::pubsub::register]` subscriptions, so call it.

## Errors

SDK calls return `Result<_, encore::Error>`: `err.code` is an `encore::ErrCode` (`NotFound`, `InvalidArgument`, `AlreadyExists`, `FailedPrecondition`, `Unavailable`, ...) and `err.message` a string. There is no built-in HTTP mapping; translate in your handlers:

```rust
match AVATARS.download(&key).await {
    Ok(bytes) => Ok(bytes),
    Err(e) if e.code == encore::ErrCode::NotFound => Err(StatusCode::NOT_FOUND),
    Err(e) => { log::error!(error:% = e; "download failed"); Err(StatusCode::INTERNAL_SERVER_ERROR) }
}
```

## Calling other services

```rust
let base = encore::services::base_url("orders"); // no trailing slash
let order = reqwest::get(format!("{base}/orders/123")).await?.error_for_status()?;
```

- The name must be a string literal, and the call must be in a service: the parser records the dependency from it.
- Append the full path the target serves, its prefix included.
- Panics if no service has that name.

Generated client, when the target declares an `openapi`, `graphql` or `grpc` API and the caller lists it in `"dependencies": ["orders"]`. `encore run` writes it to `src/encore_gen/clients/orders.rs` (or run `encore gen clients`); declare `mod encore_gen;` in the crate root:

```rust
mod encore_gen;
use encore_gen::clients::orders::Client;

let orders = Client::new(encore::services::base_url("orders"));
let order = orders.get_order("123").await?; // method per operationId, snake_case
```

The client uses `reqwest`, `serde` and `serde_json`; add them. Options: `.with_header(name, value)`, `.with_http_client(reqwest_client)`. Non-2xx returns `Error::Status { status, body }`. gRPC clients use tonic. Don't edit `encore_gen/`.

## Metadata

```rust
use encore::env::{CloudProvider, EnvironmentType};

let meta = encore::app_meta();
meta.environment.r#type == EnvironmentType::Production; // Production | Development | Ephemeral | Test
meta.environment.cloud == CloudProvider::Local;          // AWS | GCP | Azure | Encore | Local
meta.gateway("api").map(|gw| gw.base_url.as_str());
```

There is no current-request API. Pub/Sub handlers get the details on their `Message`.
