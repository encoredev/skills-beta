# Encore's API framework in Go

Module: `api.encore.dev`, errors in `api.encore.dev/errs`. It ships with the CLI next to `sdk.encore.dev` and resolves through the same generated `go.work`, so there's nothing to download; a `require api.encore.dev v0.0.0` in `go.mod` is optional. Bodies are `encoding/json/v2`, so the service needs Go 1.27.

## The program

Either let Encore generate `main`, or write it. **Generated:** a service whose root package isn't `main` (and whose `build`/`run` use the default package `.`) gets a `main` at build time, never written into the app. It makes each endpoint type with the package's `initService` if it declares one returning that type, else `new(T)`, binds them, calls `encore.Start()` and serves:

```go
// orders/orders.go: package orders, no main
type Service struct{ mu sync.Mutex; orders []*Order }

func initService() (*Service, error) { return &Service{}, nil } // optional, may be unexported
```

**Your own `main`**, needed for middleware or to construct values yourself:

```go
package main

import (
	"log"

	"api.encore.dev"
	encore "sdk.encore.dev"
)

func main() {
	srv := api.NewServer()
	api.Handle(srv, &Service{users: map[int64]*User{}}, &admin.Admin{}) // pointers for pointer receivers
	srv.Use(authMiddleware)                                               // optional
	encore.Start()                                                        // starts Pub/Sub subscriptions
	log.Fatal(api.ListenAndServe(srv))
}
```

- `ListenAndServe` listens on `ENCORE_LISTEN_ADDR`, takes part in Encore's graceful shutdown and exits the process when it's done. `*api.Server` is an `http.Handler`, so it can also be mounted in another mux.
- `api.Handle` panics on a value whose type has no endpoints, an endpoint bound twice, a bad signature, or a call after serving began. An endpoint no value serves answers 501 and is warned about at startup. Outside Encore (plain `go run`) `Handle` always panics: run with `encore run`.
- The generated `main` imports only the packages that declare endpoints. A subscription in a package none of them imports never starts (the runtime warns). It also can't serve endpoint types in a `main` package other than the root.

## Endpoints

```go
import (
	"context"
	"net/http"

	"api.encore.dev/errs"
)

type GetParams struct {
	ID     int64  `path:"id"`
	Locale string `header:"Accept-Language"`
	Fields []string                              // untagged on GET: query ?fields=a&fields=b
}

type UserResponse struct {
	User                                         // embedded: its fields are in the body
	Greeting string `header:"X-Greeting"`        // response header, not in the body
}

// Get returns a user.
//
//encore:api expose method=GET path=/users/{id}
func (s *Service) Get(ctx context.Context, p *GetParams) (*UserResponse, error) {
	u, ok := s.find(p.ID)
	if !ok {
		return nil, &errs.Error{Code: errs.NotFound, Message: "no such user"}
	}
	return &UserResponse{User: *u, Greeting: "hello"}, nil
}

type CreateResponse struct {
	User
	Status int `encore:"httpstatus"` // 0 means 200
}

//encore:api expose method=POST path=/users bodylimit=1KiB
func (s *Service) Create(ctx context.Context, p *CreateParams) (*CreateResponse, error) { /* ... */ }

// Private: other services can call it, the gateway can't.
//
//encore:api
func (s *Service) Ping(ctx context.Context) error { return nil }
```

- An endpoint is an **exported method of a named, non-generic type**, anywhere in the service's directory. Signatures: `func(ctx) error`, `func(ctx, *Req) error`, `func(ctx) (*Resp, error)`, `func(ctx, *Req) (*Resp, error)`, with `Req`/`Resp` structs. A plain function is rejected (ENC0045).
- Directive words: `expose` (default gateway) or `expose=api,admin`; `raw`; `sensitive` (traces keep only method, route, status, timing); `method=GET,POST` (`*` for all); `path=/users/{id}` (`:id` also works; last segment may be `*rest`); `name=get_user`; `bodylimit=4MiB` (`KB`, `KiB`, `MB`, `MiB`, `GB`, `GiB`, or `none`); `tag:name`.
- Defaults: name is the method's, path `/<service>.<Name>`, methods `POST` with a request, `GET` and `POST` without one, all for `raw`. v1's `public`, `private` and `auth` are rejected; `//encore:service`, `//encore:authhandler` and `//encore:middleware` too.
- Request fields: `path:"name"` (exactly one per `{name}`), `query:"name"`, `header:"Name"`, `cookie:"name"`, `"-"` drops a field. Untagged fields are query parameters in snake_case (`PageSize` is `page_size`) for GET, HEAD and DELETE, and the JSON body otherwise. Path, query, header and cookie values are strings, bools, numbers, named types of those, or `encoding.TextUnmarshaler`s (`time.Time` as RFC 3339); query and header fields may be slices. An empty value leaves the zero value; one that doesn't parse answers 400.
- Response fields: `header:"Name"` (not sent when empty), `cookie:"name"` (an `http.Cookie` or `*http.Cookie` keeps its options), `encore:"httpstatus"`. Everything else is the JSON body. A nil response answers `null`; an error-only endpoint answers 200 with no body. `X-Encore-*` headers are Encore's.
- JSON follows `encoding/json/v2` defaults: names match case-sensitively, nil slices and maps encode as `[]`/`{}`, `[]byte` is base64. What v2 refuses fails at parse time (ENC0045): `time.Duration` fields, the `format` tag option, duplicate JSON names, tags on unexported fields, structs with no exported fields.
- A body over the limit (2 MiB by default) answers 400 `invalid_argument`.
- **No request validation:** there are no validation tags. Check in the handler and return `errs.InvalidArgument`.

## Errors

```go
return nil, &errs.Error{Code: errs.InvalidArgument, Message: "a user needs a name"}
return nil, errs.B().Code(errs.PermissionDenied).Msg("not your order").Meta("order", id).Err()
return nil, errs.WrapCode(err, errs.Unavailable, "billing is down")
```

- Answers `{"code": "not_found", "message": "...", "details": ...}` with the code's status. `Meta` is never sent to clients.
- Errors that aren't `*errs.Error` keep a meaningful code: `context.Canceled`/`DeadlineExceeded`, `sql.ErrNoRows` (= `postgres.ErrNoRows`) as `not_found`, and database errors by SQLSTATE (unique violation as `already_exists`). Anything else answers 500 `internal` with no detail, and is logged. A panic is recovered, logged and answered the same way.
- `errs.HTTPError(w, err)` writes the same JSON from a raw endpoint or middleware.

## Raw endpoints

```go
//encore:api expose raw method=POST path=/webhooks/{provider}
func (s *Service) Webhook(w http.ResponseWriter, r *http.Request) {
	provider := r.PathValue("provider")
	// ... read r.Body (no body limit is applied), write w
	w.WriteHeader(http.StatusAccepted)
}
```

## Middleware and auth

There's no auth handler, and tags aren't visible at runtime. Add `net/http` middleware with `srv.Use` (own `main` only; the first added runs outermost). It runs after the request's trace and Request Context are in its context; `r.Pattern` is the matched route, as `net/http` writes it (`"GET /users/{id}"`).

Carry the caller in a Request Context field, so endpoints read it and calls to other services forward it:

```go
// auth/auth.go, a package the services share
type User struct {
	ID string `json:"id"`
}

var CurrentUser = reqctx.NewField[User]("user", reqctx.FieldConfig{}) // sdk.encore.dev/reqctx

var public = map[string]bool{"POST /login": true}

func Middleware(identify func(ctx context.Context, header string) (User, error)) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if public[r.Pattern] {
				next.ServeHTTP(w, r)
				return
			}
			user, err := identify(r.Context(), r.Header.Get("Authorization"))
			if err != nil {
				errs.HTTPError(w, &errs.Error{Code: errs.Unauthenticated, Message: "not signed in"})
				return
			}
			next.ServeHTTP(w, r.WithContext(CurrentUser.With(r.Context(), user)))
		})
	}
}
```

```go
// in an endpoint
user, ok := auth.CurrentUser.Get(ctx)
```

Clients can't set Request Context fields (Encore strips them at the edge), so a called service can trust the user it receives. To authenticate once for every service instead, use gateway middleware (`sdk.encore.dev/gateway`) that calls `gateway.Set(req, auth.CurrentUser, user)`.

## Calling other services

No typed clients are generated for `encore-api` services yet. Call the base URL over HTTP with Encore's transport, which forwards the trace and Request Context:

```go
import (
	"encoding/json"
	"net/http"
	"strconv"

	"api.encore.dev/errs"
	"sdk.encore.dev/services"
)

var client = &http.Client{Transport: services.Transport(nil)}

func getUser(ctx context.Context, id int64) (*User, error) {
	req, err := http.NewRequestWithContext(ctx, "GET", services.BaseURL("users")+"/users/"+strconv.FormatInt(id, 10), nil)
	if err != nil {
		return nil, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 300 {
		var e struct{ Message string `json:"message"` }
		_ = json.NewDecoder(resp.Body).Decode(&e)
		return nil, &errs.Error{Code: errs.HTTPStatusToCode(resp.StatusCode), Message: e.Message}
	}
	var u User
	return &u, json.NewDecoder(resp.Body).Decode(&u)
}
```

Private endpoints are reachable this way. `services.BaseURL` needs a string literal, called directly.

## Testing

Call the methods directly under `encore test`: `svc, _ := initService()` (or build the value), then `svc.Get(ctx, &GetParams{ID: 1})`. Request decoding, middleware and the error-to-HTTP mapping don't run on direct calls, so set the caller yourself (`ctx = auth.CurrentUser.With(ctx, auth.User{ID: "u1"})`) and test HTTP behavior with `encore check 'curl /users/1 -f'`. Calls to other services get 501 under `encore test`: mock them. See `encore-beta-testing`.
