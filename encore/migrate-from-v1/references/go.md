# Encore.go v1 to v2

Replace the `encore.dev` module with `sdk.encore.dev` in `go.mod` (`require sdk.encore.dev v0.1.0`; Encore resolves it to the CLI's SDK). Each service becomes a `main` package (or names one with `"run": { "type": "go", "package": "./cmd/server" }`) that serves HTTP itself and calls `encore.Start()`. See the service skill.

## API mapping

| v1 (`encore.dev/...`) | v2 (`sdk.encore.dev/...`) |
|---|---|
| `//encore:api`, `//encore:api raw` | Your router's handlers, declared to Encore with an `api` entry |
| `//encore:authhandler`, `auth.UserID()` | Framework middleware, or `gateway` Middleware setting `reqctx` fields |
| `//encore:middleware` | Framework middleware |
| `beta/errs` | `beta/errs` (same builder; `errs.HTTPError(w, err)` writes one as a response) |
| `storage/sqldb`: `sqldb.NewDatabase`, `sqldb.Named` | `storage/postgres`: `postgres.NewDatabase`, `postgres.Named` |
| `sqldb.ErrNoRows` | `postgres.ErrNoRows` |
| `db.Stdlib()`, `sqldb.Driver` | `db.ConnectionString()` with your own driver |
| `storage/objects` | `storage/objects` (same API) |
| `pubsub` | `pubsub` (handler receives `pubsub.Message[T]`) |
| `storage/cache`: `cache.NewCluster`, keyspaces | `storage/redis`: `redis.NewDatabase` with Redis commands |
| `cron`: `JobConfig{Endpoint: ...}` | `cron`: `JobConfig{Path: ...}` |
| `var secrets struct { Key string }` | `secrets.New[string]("Key")`, `NewShared`, `NewGlobal` |
| `config.Load[T]()` with CUE, `config.Value[T]` fields | `config.New[T]("name")` with TOML/JSON files, plain field types |
| `metrics.NewCounter`, `NewGauge`, groups | Same names, plus up-down counters, histograms, shared/global variants |
| `rlog` | `log/slog` |
| `encore.Meta()`, `encore.CurrentRequest()` | `env.Meta()`; no `CurrentRequest` |
| Direct calls into another service's package | `services.BaseURL("users")` over HTTP, or a generated client |
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
- **Call `encore.Start()` in `main`**, or nothing is delivered.

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
// v2: and serve POST /internal/cleanup on your router
var _ = cron.NewJob("cleanup", cron.JobConfig{Every: cron.Hour, Path: "/internal/cleanup"})
```

Secrets:

```go
// v1
var secrets struct{ StripeKey string }
stripe.Key = secrets.StripeKey
// v2
var stripeKey = secrets.New[string]("StripeKey")
key, err := stripeKey.Value()
```

Metrics now take a context: `ordersPlaced.Increment(ctx)`, `gauge.Set(ctx, v)`.

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

For typed calls, give the target an OpenAPI spec (`{ "type": "openapi", "spec": "openapi.json" }`) and list it in the caller's `dependencies` to get `encore_gen/clients/users`.
