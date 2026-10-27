# Logging, metrics and tracing in Rust

## Logging: `encore::log` + the `log` crate

```toml
log = { version = "0.4", features = ["kv"] }
```

```rust
encore::log::init(); // once, at startup: installs Encore's logger as the `log` global

log::info!(order_id = order.id, total = order.total; "order placed");

if let Err(err) = charge(&order).await {
    log::error!(error:% = err, order_id = order.id; "charge failed"); // `:%` logs via Display
    return Err(err);
}
```

- Records print in Encore's log format with the current OTel context's trace and span IDs. The logger name is the record's target (the module path by default).
- App records at `info` and above are kept; `debug!` and `trace!` are dropped.
- `init()` panics if another global logger is installed. To compose your own, wrap `encore::log::logger()` in your `log::Log`.
- There is no bridge for the `tracing` crate's events; log through `log`.

## Metrics: `encore::metrics`

```rust
use encore::metrics::{Counter, CounterGroup, Gauge, Histogram, Labels, UpDownCounter};

/// Orders placed.                                         // doc comment = description
static ORDERS_PLACED: Counter = Counter::new("orders_placed").unit("{order}");
static QUEUED_JOBS: UpDownCounter = UpDownCounter::new("queued_jobs");
static OPEN_CONNECTIONS: Gauge = Gauge::new("open_connections");
static UPLOAD_SIZE: Histogram =
    Histogram::new("upload_size").unit("By").buckets(&[1024.0, 65536.0, 1048576.0]);

ORDERS_PLACED.increment();       // synchronous
QUEUED_JOBS.add(batch.len() as i64);
OPEN_CONNECTIONS.set(pool_size as f64);
UPLOAD_SIZE.record(file.len() as f64);

#[derive(Labels)]
struct Outcome { method: &'static str, status: u16 }
static REQUESTS: CounterGroup<Outcome> = CounterGroup::new("requests_handled");
REQUESTS.with(&Outcome { method: "GET", status: 200 }).increment();
```

- Value types: `Counter<u64|f64>`, `UpDownCounter<i64|f64>`, `Gauge<f64|i64|u64>`, `Histogram<f64>` (first is the default).
- Groups: `CounterGroup`, `UpDownCounterGroup`, `GaugeGroup`, `HistogramGroup`.
- Shared and global variants: `SharedCounter`, `GlobalHistogram`, `SharedCounterGroup`, and so on.
- Bind by name: `static SERVED: CounterGroup<Served> = CounterGroup::named("greetings.served");`
- Pass to other code with `METRIC.reference()`.
- Service-owned metrics export as `<service>.<name>`. Under `encore test` metrics record nothing.

OpenTelemetry instruments: install Encore's meter provider as the global, then use the OTel API (re-exported as `encore::opentelemetry`, version 0.29):

```rust
encore::opentelemetry::global::set_meter_provider(encore::metrics::meter_provider());
let views = encore::opentelemetry::global::meter("api").u64_counter("page_views").build();
views.add(1, &[encore::opentelemetry::KeyValue::new("page", "home")]);
```

## Tracing: Encore's OpenTelemetry providers

Encore records its own spans through its providers and leaves the OTel globals to you. Database, Redis, bucket and Pub/Sub calls parent on `Context::current()`, so run each request under a span that continues the gateway's `traceparent`:

```rust
use axum::{extract::Request, middleware::Next, response::Response};
use encore::opentelemetry::trace::{FutureExt, SpanKind, TraceContextExt, Tracer};
use encore::opentelemetry::Context;

async fn trace_request(req: Request, next: Next) -> Response {
    let headers = req.headers();
    let header = |name: &str| headers.get(name).and_then(|v| v.to_str().ok());
    let parent = match header("traceparent")
        .and_then(|tp| encore::telemetry::parse_traceparent(tp, header("tracestate")))
    {
        Some(remote) => Context::new().with_remote_span_context(remote),
        None => Context::new(),
    };
    let tracer = encore::telemetry::tracer("api");
    let span = tracer
        .span_builder(format!("{} {}", req.method(), req.uri().path()))
        .with_kind(SpanKind::Server)
        .start_with_context(&tracer, &parent);
    next.run(req).with_context(parent.with_span(span)).await
}

let app = Router::new().route(/* ... */).layer(axum::middleware::from_fn(trace_request));
```

Own spans (ended on drop):

```rust
let tracer = encore::telemetry::tracer("orders");
let span = tracer.start("price order");
let price = price_order(&order).with_context(Context::current_with_span(span)).await?;
```

For libraries that use the globals: `encore::opentelemetry::global::set_tracer_provider(encore::telemetry::tracer_provider())`. The `tracing` crate's spans reach Encore through `tracing-opentelemetry` with `encore::telemetry::tracer(..)`, but it needs a `PreSampledTracer` wrapper; see `tests/rs/axum-hello-world/src/otel.rs` in the v2-encore repo.
