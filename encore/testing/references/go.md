# Testing in Go

`encore test` runs each service's `test` strategy. Go's default is `go test ./...` in the service directory, with `sdk.encore.dev` resolved to the CLI's SDK. Use the standard `testing` package.

```bash
encore test                          # every service
encore test --services api           # one service
GOFLAGS="-v -run=TestInsert" encore test --services api   # pass go test flags
```

Set it in `encore.service.json`: `"test": { "type": "go", "package": "./internal/..." }` or `{ "type": "custom", "cmd": [...] }`. An explicit command (`encore test -- go test ./...`) runs without Encore's `go.work`, so from the app root it fails to find the module; prefer the strategy.

## Database tests

```go
// users/users_test.go
func TestInsert(t *testing.T) {
	ctx := context.Background()
	if _, err := db.Exec(ctx, "INSERT INTO users (email, name) VALUES ($1, $2)", "ada@x.com", "Ada"); err != nil {
		t.Fatal(err)
	}
	var name string
	if err := db.QueryRow(ctx, "SELECT name FROM users WHERE email = $1", "ada@x.com").Scan(&name); err != nil {
		t.Fatal(err)
	}
}
```

Isolation is per test process: each package's test binary gets its own migrated database (and its own bucket prefix, topics and Redis namespace). Tests in one package share them, so with `t.Parallel()` or shared tables use unique values per test or clean up. `env.Meta().Environment.Type` is `env.EnvTest`.

Config files apply in tests, except `*.user.config.*` overrides.

## HTTP route tests

Build the router in a function that doesn't listen, and test it with `net/http/httptest`:

```go
// main.go: func newMux() *http.ServeMux { ... }; main() serves newMux()
func TestMissingUser(t *testing.T) {
	rec := httptest.NewRecorder()
	newMux().ServeHTTP(rec, httptest.NewRequest("GET", "/users/999999", nil))
	if rec.Code != http.StatusNotFound {
		t.Fatalf("got %d", rec.Code)
	}
}
```

## Other services

`services.BaseURL("orders")` returns a URL, but requests get `501 Not Implemented`. Mock them, for example with an `http.Client` whose transport you control, passed to a generated client with `orders.WithHTTPClient(...)`, or with `httptest.NewServer`.

## Pub/Sub

Subscriptions deliver in tests only after `encore.Start()`. Either start them in `TestMain`:

```go
func TestMain(m *testing.M) {
	encore.Start()
	os.Exit(m.Run())
}
```

or call the handler function directly with a message you build:

```go
err := sendConfirmation(ctx, pubsub.Message[*OrderPlaced]{Data: &OrderPlaced{OrderID: "1"}})
```

There's no v1 `et` package: no `et.MockEndpoint`, `et.NewTestDatabase` or `et.Topic`.
