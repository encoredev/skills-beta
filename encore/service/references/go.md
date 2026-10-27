# Go services

## Install the SDK

The module is `sdk.encore.dev`. Require it in the service's `go.mod`; the version there doesn't matter:

```
module orders

go 1.27.0

require sdk.encore.dev v0.1.0
```

`encore run`, `test`, `check` and `build` resolve `sdk.encore.dev` to the SDK that ships with the CLI, through a `go.work` they generate, so the SDK always matches the CLI. The module isn't published: a plain `go build` or `go mod tidy` outside Encore can't fetch it. `encore service new` writes the `go.mod` for you.

## Run strategy

Go has a launch convention, so `run` can be left out. These are equivalent:

```json
"run": { "type": "go" }
"run": { "type": "go", "package": "." }
```

`package` is the main package, relative to the service root (e.g. `"./cmd/server"`). Encore runs `go build` as a build step, then the binary. Anything else:

```json
"run": { "type": "custom", "cmd": ["./bin/server"], "build": ["make", "server"] }
```

## Server

Keep the router (`net/http`, chi, gin, echo). Listen on `ENCORE_LISTEN_ADDR`, call `encore.Start()`, and hand the server's drain to Encore:

```go
package main

import (
	"errors"
	"log/slog"
	"net/http"
	"os"

	encore "sdk.encore.dev"
	"sdk.encore.dev/shutdown"
)

func main() {
	defer shutdown.Exit() // finishes the shutdown and exits with its code, however main ends
	encore.Start()        // starts Pub/Sub subscriptions; nothing is delivered until it's called

	mux := http.NewServeMux()
	// ... routes ...

	srv := &http.Server{Addr: os.Getenv("ENCORE_LISTEN_ADDR"), Handler: mux}
	shutdown.OnShutdown(srv.Shutdown) // Encore closes infrastructure only after this returns
	if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
		slog.Error("serving failed", "err", err)
	}
}
```

- **`encore.Start()` is required for subscriptions.** Unlike TypeScript, declaring one doesn't start it, and Encore logs a warning when an app with subscriptions never calls it. Databases, buckets and Redis open on first use and need no start.
- On `SIGTERM`/`SIGINT`, Encore stops fetching Pub/Sub messages, waits for in-flight handlers and `OnShutdown` drains (each gets `shutdown.DrainContext()`), closes infrastructure, flushes telemetry and exits 0 (1 after the deadline). A second signal stops the waiting.
- Other hooks in `sdk.encore.dev/shutdown`: `Started()` and `Cleanup()` channels, `CurrentPhase()` for readiness checks, `Begin(err)` to start a shutdown yourself, `IgnoreSignals()`, `Wait()`, `Finish()`.

## Calling other services

```go
import "sdk.encore.dev/services"

var client = &http.Client{Transport: services.Transport(nil)} // forwards Request Context to the app's services

req, err := http.NewRequestWithContext(ctx, "GET", services.BaseURL("orders")+"/orders/123", nil)
if err != nil {
	return err
}
resp, err := client.Do(req)
```

- `BaseURL` has no trailing slash; append a path starting with `/`. The name must be a string literal and the call must be direct: it's how Encore records the dependency.
- Make the request with the incoming request's context, or its trace and Request Context are lost.
- With tracing, wrap the traced transport: `services.Transport(otelhttp.NewTransport(http.DefaultTransport))`.

Generated client, when the target declares an `openapi`, `graphql` or `grpc` API and the caller lists it in `"dependencies": ["orders"]`. `encore run` writes it to `encore_gen/clients/orders/orders.go` (or run `encore gen clients`), importable from the caller's module:

```go
import "api/encore_gen/clients/orders" // <your module>/encore_gen/clients/<service>

oc := orders.New(services.BaseURL("orders"), orders.WithHTTPClient(client))
order, err := oc.GetOrder(ctx, "123") // method per operationId; non-2xx returns *orders.APIError
```

Options: `WithHTTPClient(c)`, `WithHeader(key, value)` for every request. Don't edit `encore_gen/`.

## Metadata

```go
import "sdk.encore.dev/env"

meta := env.Meta()
meta.Environment.Type  // env.EnvProduction, EnvDevelopment, EnvEphemeral, EnvTest
meta.Environment.Cloud // env.CloudAWS, CloudGCP, CloudAzure, EncoreCloud, CloudLocal
meta.Gateways["api"]   // *env.Gateway with APIBaseURL; empty in a process that serves no gateway
```

Locally, `encore run` reports `EnvDevelopment` with `CloudLocal`; `encore test` reports `EnvTest`.

Request Context: typed values the gateway (or a caller) attaches to a request, read with `field.Get(r.Context())`. Declare with `reqctx.NewField[T]("user", reqctx.FieldConfig{})` from `sdk.encore.dev/reqctx`; bind one declared elsewhere with `reqctx.Named[T]("user")`. A server receives them once `tracing.Propagator()` is installed (see the observability skill) or its handler is wrapped with `reqctx.Handler`.
