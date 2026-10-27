# Secrets in Go

Package: `sdk.encore.dev/secrets`.

```go
import "sdk.encore.dev/secrets"

var stripeKey = secrets.New[string]("StripeKey")          // the declaring service's own
var signingKey = secrets.NewShared[string]("SigningKey")  // one app-wide value; readers are tracked
var sentryDSN = secrets.NewGlobal[string]("SentryDSN")    // one app-wide value; every service, untracked

key, err := stripeKey.Value()  // fails if unset
dsn := sentryDSN.StringValue() // the stored string, "" if unset
```

- The type argument decodes the value: `string` and `[]byte` as stored, anything else from JSON (`secrets.New[Credentials]("DBCreds")`).
- Values can change while the app runs. Read where you use it instead of caching for the process lifetime.
- Locally, an unset secret makes `Value()` return an error and `StringValue()` return `""`. Unlike TypeScript there's no silent empty value from `Value()`.
- `stripeKey.String()` returns `Secret(StripeKey: *********)`, never the value.

A secret can only be used by calling its methods. Passing it to a function (even one taking `*secrets.Secret[string]`), assigning it, or handing it to `fmt.Sprint` is a parse error, and there's no reference type. Read the value and pass that.

v1's `var secrets struct { StripeKey string }` is gone; declare each secret with `secrets.New`.
