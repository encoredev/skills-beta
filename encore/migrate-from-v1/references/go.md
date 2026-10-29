# Encore.go v1 to v2

Replace the `encore.dev` module with `sdk.encore.dev` in `go.mod` (`require sdk.encore.dev v0.1.0`; Encore resolves it to the CLI's SDK, any version). Go 1.27 is required.

Two ways to serve each service:

- **Keep the `//encore:api` endpoints** on Encore's API framework (`"type": "encore-api"`, see `encore-beta-api`'s Go reference). Endpoints become methods, path parameters move into the request struct, and Encore generates `main` unless you write one.
- **Port to `net/http` or a router**: the service becomes a `main` package (or names one with `"run": { "type": "go", "package": "./cmd/server" }`) that listens on `ENCORE_LISTEN_ADDR`, calls `encore.Start()` and declares its routes with an `api` entry. See the service skill.

## API mapping

| v1 (`encore.dev/...`) | v2 |
|---|---|
| `//encore:api` functions, `raw` too | Methods with `//encore:api` (`api.encore.dev`), or your router's handlers with an `api` entry |
| `//encore:api public`, `private`, `auth` | `expose`, or nothing for private; `auth` has no equivalent |
| `//encore:service` struct, `initService` | The directive goes; `initService` still builds the value for a generated `main` |
| `//encore:authhandler`, `auth.UserID()` | Middleware (`srv.Use`) or `gateway` Middleware setting a `reqctx` field |
| `//encore:middleware` | `srv.Use` net/http middleware, or your framework's |
| `beta/errs` | `api.encore.dev/errs` (same `Error`, codes and builder; `errs.HTTPError(w, err)` writes one from any handler). `sdk.encore.dev` has no `errs`. |
| `storage/sqldb`: `sqldb.NewDatabase`, `sqldb.Named` | `sdk.encore.dev/storage/postgres`: `postgres.NewDatabase`, `postgres.Named` |
| `sqldb.ErrNoRows` | `postgres.ErrNoRows` (it is `sql.ErrNoRows`) |
| `db.Stdlib()`, `sqldb.Driver` | `db.ConnectionString()` with your own driver |
| `storage/objects` | `storage/objects` (same API) |
| `pubsub` | `pubsub` (handler receives `pubsub.Message[T]`) |
| `storage/cache`: `cache.NewCluster`, keyspaces | `storage/redis`: `redis.NewDatabase` with Redis commands |
| `cron`: `JobConfig{Endpoint: ...}` | `cron`: `JobConfig{Path: ...}` |
| `var secrets struct { Key string }` (app-wide) | `secrets.NewShared[string]("Key")` or `NewGlobal`; `secrets.New` is per service |
| `config.Load[T]()` with CUE, `config.Value[T]` fields | `config.New[T]("name")` with TOML/JSON files, plain field types |
| `metrics.NewCounter`, `NewGauge`, groups | Same names, plus up-down counters, histograms, shared/global variants |
| `rlog` | `log/slog` |
| `encore.Meta()`, `encore.CurrentRequest()` | `env.Meta()` (`sdk.encore.dev/env`); no `CurrentRequest` |
| Direct calls into another service's package | `services.BaseURL("users")` over HTTP, or a client generated from an OpenAPI/GraphQL/gRPC spec |
| `et` (`MockEndpoint`, `NewTestDatabase`, `Topic`) | None; tests get per-process infrastructure (see the testing skill) |

## Changes per API

Databases: same calls, new package.

```go
// v1
var db = sqldb.NewDatabase("users", sqldb.DatabaseConfig{Migrations: "./migrations"})
// v2
var db = postgres.NewDatabase("users", postgres.DatabaseConfig{Migrations: "./migrations"})
```

`Exec`, `Query`, `QueryRow`, `Begin` keep their signatures. Pass `db.Read()`/`db.Write()` (a `postgres.Queryable`) to helpers; passing `db` itself is a parse error.

Pub/Sub:

```go
// v1
var _ = pubsub.NewSubscription(Signups, "send-welcome", pubsub.SubscriptionConfig[*SignupEvent]{
	Handler: func(ctx context.Context, e *SignupEvent) error { return welcome(ctx, e.UserID) },
	RetryPolicy: &pubsub.RetryPolicy{MaxRetries: 10},
})
// v2
var _ = pubsub.NewSubscription(Signups, "send-welcome", pubsub.SubscriptionConfig[*SignupEvent]{
	Handler: func(ctx context.Context, msg pubsub.Message[*SignupEvent]) error { return welcome(ctx, msg.Data.UserID) },
	MaxRetries: 10, // MinBackoff, MaxBackoff are top-level fields too
})
```

- `TopicConfig{DeliveryGuarantee: pubsub.AtLeastOnce}` becomes `pubsub.TopicConfig{}`; exactly-once is `ExactlyOnce: true`.
- `OrderingAttribute` becomes `pubsub.NewOrderedTopic`, with the key passed to `Publish`.
- `pubsub.MethodHandler` is gone; use a plain function.
- **`main` must call `encore.Start()`**, or nothing is delivered (a generated `main` does). A subscription must be in a package `main` imports.

Caching: no clusters or keyspaces; build keys yourself and encode structs (e.g. JSON).

```go
// v1
var cluster = cache.NewCluster("sessions", cache.ClusterConfig{EvictionPolicy: cache.AllKeysLRU})
var tokens = cache.NewStringKeyspace[string](cluster, cache.KeyspaceConfig{KeyPattern: "session/:key", DefaultExpiry: cache.ExpireIn(time.Hour)})
err := tokens.Set(ctx, id, token)
// v2
var sessions = redis.NewDatabase("sessions", redis.DatabaseConfig{})
err := sessions.String.SetEx(ctx, "session/"+id, token, 3600)
```

`cache.Miss` becomes `redis.ErrNil`.

Cron:

```go
// v1
var _ = cron.NewJob("cleanup", cron.JobConfig{Every: cron.Hour, Endpoint: Cleanup})
// v2: and serve POST /internal/cleanup, a private endpoint or a router route
var _ = cron.NewJob("cleanup", cron.JobConfig{Every: cron.Hour, Path: "/internal/cleanup"})
```

Secrets:

```go
// v1
var secrets struct{ StripeKey string }
stripe.Key = secrets.StripeKey
// v2: v1 secrets were app-wide, and Encore Cloud stores app-wide secrets only
var stripeKey = secrets.NewShared[string]("StripeKey")
key, err := stripeKey.Value()
```

Metrics now take a context: `ordersPlaced.Increment(ctx)`, `gauge.Set(ctx, v)`.

## Keeping an endpoint (Encore's API framework)

```go
// v1
//encore:api public method=GET path=/users/:id
func Get(ctx context.Context, id int64) (*User, error) { ... }

// v2: service "users" with "type": "encore-api"
type Service struct{}

type GetParams struct {
	ID int64 `path:"id"`
}

//encore:api expose method=GET path=/users/{id}
func (s *Service) Get(ctx context.Context, p *GetParams) (*User, error) {
	var u User
	err := db.QueryRow(ctx, "SELECT id, name FROM users WHERE id = $1", p.ID).Scan(&u.ID, &u.Name)
	if errors.Is(err, postgres.ErrNoRows) {
		return nil, &errs.Error{Code: errs.NotFound, Message: "user not found"} // api.encore.dev/errs
	}
	return &u, err
}
```

- Every endpoint becomes an exported method; one value type per service is enough. Path parameters become `path:"name"` fields of the request struct, so a v1 endpoint with path parameters and a body merges both into one struct.
- `public` becomes `expose`. Endpoints with `auth` need middleware (see `encore-beta-api`).
- Returning `postgres.ErrNoRows` unwrapped also answers 404 `not_found`.

## Porting an endpoint (`net/http`)

```go
// v1
//encore:api public method=GET path=/users/:id
func Get(ctx context.Context, id int64) (*User, error) {
	var u User
	err := db.QueryRow(ctx, "SELECT id, name FROM users WHERE id = $1", id).Scan(&u.ID, &u.Name)
	if errors.Is(err, sqldb.ErrNoRows) {
		return nil, &errs.Error{Code: errs.NotFound, Message: "user not found"}
	}
	return &u, err
}

// v2: service "users" with { "type": "http", "prefix": "/users", "expose": true }
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseInt(r.PathValue("id"), 10, 64)
	if err != nil {
		http.Error(w, "invalid id", http.StatusBadRequest)
		return
	}
	var u User
	err = db.QueryRow(r.Context(), "SELECT id, name FROM users WHERE id = $1", id).Scan(&u.ID, &u.Name)
	if errors.Is(err, postgres.ErrNoRows) {
		http.Error(w, "user not found", http.StatusNotFound)
		return
	} else if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	json.NewEncoder(w).Encode(u)
})
```

## Replacing direct service calls

```go
// v1
u, err := users.Get(ctx, id)
// v2
req, _ := http.NewRequestWithContext(ctx, "GET", services.BaseURL("users")+"/users/"+strconv.FormatInt(id, 10), nil)
resp, err := client.Do(req) // client: &http.Client{Transport: services.Transport(nil)}
```

No clients are generated for `encore-api` services; `encore-beta-api` has a call helper that turns error responses back into `*errs.Error`. For typed calls to a router-based service, give the target an OpenAPI spec (`{ "type": "openapi", "spec": "openapi.json" }`) and list it in the caller's `dependencies` to get `encore_gen/clients/users`.
