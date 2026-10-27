---
name: encore-beta-observability
description: Logging, metrics and distributed tracing in an Encore v2 (beta) app, in TypeScript, Python, Go or Rust. Covers structured logs correlated with traces, counters/gauges/histograms with labels, registering Encore's OpenTelemetry tracer and meter providers, framework instrumentation, cross-service trace propagation, and the `encore logs`, `encore trace` and `encore metrics` commands.
when_to_use: >-
  User is in an Encore v2 app and wants to add or improve logs, count or measure something, see their framework's requests or outgoing HTTP calls in traces, connect traces across services, add custom spans, or read logs, traces or metrics locally. Also use when requests show up untraced or logs have no trace IDs. Trigger phrases: "logging", "log fields", "metrics", "counter", "histogram", "tracing", "OpenTelemetry", "OTel", "span", "traceparent", "encore logs", "encore trace", "dev dashboard", "instrumentation".
---

# Observability in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

v2 telemetry is OpenTelemetry. Without any setup, Encore traces what passes through it: gateway requests, calls between services, and every SDK operation (queries, publishes, message handling, bucket and Redis calls). **Your framework's routing, outgoing HTTP calls, other database clients and your own spans appear only after you register Encore's providers** (Python's `encore.run(app)` does it for you). Encore never installs OpenTelemetry globals on its own, so it won't fight an existing setup.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`).

## Logging

- Use the SDK's structured logger (or Python's `logging` with `encore.logs.ConsoleHandler`) so each record carries fields and the active trace/span IDs. Plain `console.log`/`print` output is collected but uncorrelated.
- **Constant messages, details in fields:** `log.info("order placed", { orderId })`, not an interpolated string. Constant messages can be searched and counted.
- Log errors with the error object so type and stack are recorded. Log a failure once, where it's handled.
- `ERROR` for operations that failed unexpectedly; expected outcomes (declined payment, failed validation) are not errors. `WARNING` for recovered problems. `INFO` for meaningful, low-volume events.
- Don't log what the trace already shows ("querying database"). Log business events and decisions with their IDs.
- **Never log** passwords, tokens, API keys, authorization headers, secret values or payment details. Prefer internal IDs over emails. Encore does not redact log fields.

## Metrics

- Declare at module scope with a literal name. Kinds: `Counter` (only up), `UpDownCounter`, `Gauge` (last value), `Histogram` (distributions). Units are UCUM (`"s"`, `"By"`, `"{request}"`).
- A metric belongs to the declaring service and is exported as `<service>.<name>`. Shared/global variants exist for metrics several services record to.
- Labels: one time series per combination. Use low-cardinality values (status, region), **never user IDs or URLs**.
- When you only need how often something happens, use a metric, not a log line.

## Tracing

- Register Encore's tracer and meter providers as the OpenTelemetry globals at startup, **before** importing the modules to instrument, then add your framework's instrumentation package.
- Calls between services join one trace when the caller sends the W3C `traceparent` header. HTTP client instrumentation does it; otherwise inject it yourself. Traces cross languages.
- Encore decides sampling. Deployed environments redact endpoints and fields marked sensitive; `encore run --redact` shows the redacted form locally.

## Viewing locally

```bash
encore logs                          # latest session; works when the app isn't running
encore logs --errors --svc orders --since 10m
encore trace list
encore trace get <trace-id>
encore metrics list                  # add --watch to refresh
encore metrics get shop.orders_placed --since 5m
encore dashboard                     # local dev dashboard: traces, logs, metrics, architecture
```

`ENCORE_LOG` sets the lowest level written (`trace` ... `fatal`, or `off`) in TypeScript.
