# Logging, metrics and tracing in TypeScript

## Logging: `@encore.dev/sdk/log`

```typescript
import log, { LogLevel } from "@encore.dev/sdk/log";

log.info("order placed", { orderId: order.id, total: order.total });

try {
  await charge(order);
} catch (err) {
  log.error(err, "charge failed", { orderId: order.id }); // warn/error/fatal take the error first
  throw err;
}

const ordersLog = log.withName("orders");          // names the logger
const reqLog = ordersLog.with({ customerId });     // adds fields to every record
const quiet = log.withLevel(LogLevel.Warn);
```

Levels: `trace`, `debug`, `info`, `warn`, `error`, `fatal`; `ENCORE_LOG` sets the lowest logged (`trace` by default, `off` for none). `console.log`, pino and winston output still appears, but as plain text with no fields or trace IDs.

## Metrics: `@encore.dev/sdk/metrics`

```typescript
import { Counter, Gauge, Histogram, UpDownCounter, CounterGroup } from "@encore.dev/sdk/metrics";

/** Orders placed. */                                      // JSDoc = description
export const ordersPlaced = new Counter("orders_placed", { unit: "{order}" });
export const queuedJobs = new UpDownCounter("queued_jobs");
export const openConnections = new Gauge("open_connections");
export const uploadSize = new Histogram("upload_size", { unit: "By", buckets: [1024, 65536, 1048576] });

ordersPlaced.increment();          // synchronous
queuedJobs.add(batch.length);
openConnections.set(pool.size);
uploadSize.record(file.length);

interface Outcome { method: string; status: number }
export const requests = new CounterGroup<Outcome>("requests_handled");
requests.with({ method: "GET", status: 200 }).increment();
```

Groups: `CounterGroup`, `UpDownCounterGroup`, `GaugeGroup`, `HistogramGroup`. Shared and global variants: `SharedCounter`, `GlobalHistogram`, `SharedCounterGroup`, and so on. Bind by name: `SharedCounterGroup.named<Served>("greetings.served")`. Pass to other code with `metric.ref()`.

OpenTelemetry instruments: `meterProvider().getMeter("payments")` from `@encore.dev/sdk/metrics`.

## Tracing: register Encore's OpenTelemetry providers

Encore's API framework (`@encore.dev/api`) records a server span per request and continues the caller's trace by itself. Register the providers for your own spans and for instrumentations (other frameworks, HTTP clients, ORMs).

```typescript
// otel.ts: import it FIRST in the entry point (import "./otel";)
import { meterProvider, tracerProvider } from "@encore.dev/sdk";
import { context, metrics, propagation, trace } from "@opentelemetry/api";
import { AsyncLocalStorageContextManager } from "@opentelemetry/context-async-hooks";
import { W3CTraceContextPropagator } from "@opentelemetry/core";
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { ExpressInstrumentation } from "@opentelemetry/instrumentation-express";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";

context.setGlobalContextManager(new AsyncLocalStorageContextManager());
propagation.setGlobalPropagator(new W3CTraceContextPropagator());
trace.setGlobalTracerProvider(tracerProvider());
metrics.setGlobalMeterProvider(meterProvider());

registerInstrumentations({ instrumentations: [new HttpInstrumentation(), new ExpressInstrumentation()] });
```

```bash
npm install @opentelemetry/api @opentelemetry/context-async-hooks @opentelemetry/core \
  @opentelemetry/instrumentation @opentelemetry/instrumentation-http @opentelemetry/instrumentation-express
```

Swap in the instrumentations for your stack (Fastify, Hono, NestJS, undici, `pg`, Prisma, ...).

**Bun:** instrumentations only patch modules loaded through CommonJS:

```typescript
import { createRequire } from "node:module";
const express: typeof import("express") = createRequire(import.meta.url)("express");
```

Bun's `fetch` has no instrumentation. Propagate trace context on outgoing calls by hand:

```typescript
const headers: Record<string, string> = {};
propagation.inject(context.active(), headers);
await fetch(new URL("orders/123", serviceUrl("orders")), { headers });
```

Own spans:

```typescript
const tracer = trace.getTracer("orders");
await tracer.startActiveSpan("price order", async (span) => {
  try { return await priceOrder(order); } finally { span.end(); }
});
```
