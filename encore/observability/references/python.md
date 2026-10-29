# Logging, metrics and tracing in Python

## Logging: standard `logging` + `encore.logs.ConsoleHandler`

```python
# main.py
import logging
from encore.logs import ConsoleHandler

logging.basicConfig(level=logging.INFO, handlers=[ConsoleHandler()])
```

```python
log = logging.getLogger(__name__)

log.info("order created", extra={"order_id": order.id, "total": 14900})

try:
    await charge_customer(customer_id, amount)
except PaymentError:
    log.exception("payment authorization failed", extra={"customer_id": customer_id})
    raise
```

- `extra` keys can't clash with `LogRecord` attributes (`name`, `message`, `args`, ...).
- `ConsoleHandler` ignores formatters; it writes Encore's JSON line format with trace and span IDs.
- `print()` output is collected as plain text.

## Metrics: `from encore import metrics`

```python
orders_placed = metrics.Counter("orders_placed", description="Orders placed.")
queued = metrics.UpDownCounter("jobs_queued", unit="{job}")
oven = metrics.Gauge("oven_temperature", unit="Cel")
latency = metrics.Histogram("order_latency", unit="s", buckets=[0.1, 1, 10])   # literal list

orders_placed.increment()   # add() ignores negative amounts on a Counter
queued.add(12)
oven.set(180.5)
latency.record(0.25)
```

Labels (`TypedDict` or frozen dataclass, named directly in the subscript):

```python
class Outcome(TypedDict):
    method: str
    status: int

requests_handled = metrics.CounterGroup[Outcome]("requests_handled")
requests_handled.with_labels({"method": "GET", "status": 200}).increment()
```

Label values: `str`, `int`, `bool`. Shared/global: `metrics.SharedCounter(...)`, `metrics.GlobalHistogram(...)`; declare once and import the declaration. Bind a shared or global metric declared in another language by name: `metrics.Counter.named("cache.hits")`, `metrics.CounterGroup[Outcome].named(...)` (no options; it takes the declaration's). Pass to other code with `metric.ref()`. Under `encore test`, metrics are recorded but not exported.

## Tracing

**With `encore.run(app)`** you get a server span per request, named after the matched route, and Encore's tracer provider installed as the OpenTelemetry global (unless you set one). Nothing to set up for request tracing. `encore.run` turns FastAPI's own spans off for its requests; `encore.run(app, server_spans=False)` leaves them to FastAPI instead.

**With your own server**, add Encore's middleware, which records the same server span and installs the tracer provider:

```python
from encore.server import asgi, wsgi

app.add_middleware(asgi.Middleware)   # FastAPI, Starlette
app = wsgi.Middleware(app)            # Flask, Django's WSGI app (spans aren't named after routes)
```

For instrumentation packages (`opentelemetry-instrumentation-httpx`, `-fastapi`, ...) and their metrics, install Encore's providers as the globals:

```python
# main.py: import encore (directly or via these) before instrumentation packages
from opentelemetry import metrics as otel_metrics, trace
from encore.metrics import meter_provider
from encore.tracing import tracer_provider

trace.set_tracer_provider(tracer_provider())
otel_metrics.set_meter_provider(meter_provider())
```

Encore's middleware works beside `FastAPIInstrumentor`. Importing `encore` sets `OTEL_SEMCONV_STABILITY_OPT_IN=http`, which instrumentation packages read at import, so import `encore` first.

Own spans:

```python
tracer = trace.get_tracer(__name__)

async def get_user(user_id: int):
    with tracer.start_as_current_span("load user") as span:
        span.set_attribute("user.id", user_id)
        return await db.query("SELECT * FROM users WHERE id = $1", user_id).one()
```

`start_as_current_span` also works as a decorator on `async` functions.
