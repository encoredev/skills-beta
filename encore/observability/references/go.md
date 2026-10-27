# Logging, metrics and tracing in Go

## Logging: `log/slog`

The SDK installs its handler as slog's default when it loads, so `slog.Info` and the standard `log` package print in Encore's format with no setup. Use the `Context` variants so records carry the trace and span IDs:

```go
slog.InfoContext(ctx, "order placed", "order_id", order.ID, "total", order.Total)

if err := charge(ctx, order); err != nil {
	slog.ErrorContext(ctx, "charge failed", "order_id", order.ID, "err", err)
	return err
}

ordersLog := slog.With("customer_id", customerID) // fields on every record
```

A named logger, or other options:

```go
import "sdk.encore.dev/env"

slog.SetDefault(slog.New(env.NewLogHandler(&env.LogHandlerOptions{
	Name:      "orders",       // instrumentation scope; default is the main module's path
	Level:     slog.LevelDebug, // default Info
	AddSource: true,
})))
```

`ENCORE_LOG_FORMAT` (`pretty`, `json`, `gcp`, `off`) overrides the format. There's no `rlog` in v2: use slog.

## Metrics: `sdk.encore.dev/metrics`

```go
import "sdk.encore.dev/metrics"

// Orders placed.                                         (doc comment = description)
var ordersPlaced = metrics.NewCounter[uint64]("orders_placed", metrics.CounterConfig{Unit: "{order}"})
var queuedJobs = metrics.NewUpDownCounter[int64]("queued_jobs", metrics.UpDownCounterConfig{})
var openConnections = metrics.NewGauge[int64]("open_connections", metrics.GaugeConfig{})
var uploadSize = metrics.NewHistogram("upload_size", metrics.HistogramConfig{Unit: "By", Buckets: []float64{1024, 65536, 1048576}})

ordersPlaced.Increment(ctx) // every call takes ctx: sampled spans become exemplars
queuedJobs.Add(ctx, int64(len(batch)))
openConnections.Set(ctx, int64(pool.Size()))
uploadSize.Record(ctx, float64(len(file)))

type Outcome struct {
	Method string // label "method"; fields become snake_case labels
	Status int
}

var requests = metrics.NewCounterGroup[Outcome, uint64]("requests_handled", metrics.CounterConfig{})
requests.With(Outcome{Method: "GET", Status: 200}).Increment(ctx)
```

Value types: counters `uint64`/`float64`, up-down counters `int64`/`float64`, gauges `int64`/`uint64`/`float64`, histograms `float64`. Labels are strings, bools or integers.

Groups: `NewCounterGroup`, `NewUpDownCounterGroup`, `NewGaugeGroup`, `NewHistogramGroup`. Shared and global variants: `NewSharedCounter`, `NewGlobalHistogram`, `NewSharedCounterGroup`, and so on. Bind by name: `metrics.NamedCounterGroup[Served, uint64]("greetings.served")`. Pass to other code with `metric.Ref()`.

A service's metric is exported as `<service>.<name>`. OpenTelemetry instruments: `metrics.MeterProvider().Meter("payments")`, or the global meter once it's installed (below).

## Tracing: install Encore's OpenTelemetry providers

The runtime installs no global provider or propagator. Do it first thing in `main`:

```go
import (
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"

	"sdk.encore.dev/metrics"
	"sdk.encore.dev/services"
	"sdk.encore.dev/tracing"
)

func main() {
	otel.SetTracerProvider(tracing.TracerProvider())
	otel.SetMeterProvider(metrics.MeterProvider())
	otel.SetTextMapPropagator(tracing.Propagator()) // W3C trace context + Request Context

	mux := http.NewServeMux()
	// ... routes ...
	srv := &http.Server{Addr: os.Getenv("ENCORE_LISTEN_ADDR"), Handler: otelhttp.NewHandler(mux, "server")}
	// ...
}

var client = &http.Client{Transport: services.Transport(otelhttp.NewTransport(http.DefaultTransport))}
```

Add the instrumentation for your stack (chi, gin, echo, pgx, go-redis, ...). Database, Redis, bucket and Pub/Sub calls made through the SDK are traced already, as children of the span in the `ctx` you pass.

Own spans:

```go
ctx, span := otel.Tracer("orders").Start(ctx, "price order")
defer span.End()
```
