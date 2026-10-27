# Secrets in Rust

Module: `encore::secrets`.

```rust
use encore::secrets::{GlobalSecret, Json, Secret, SharedSecret};

static STRIPE_KEY: Secret = Secret::new("StripeKey");               // the declaring service's own
static SIGNING_KEY: SharedSecret<Vec<u8>> = SharedSecret::new("SigningKey"); // one app-wide value; readers are tracked
static SENTRY_DSN: GlobalSecret = GlobalSecret::new("SentryDSN");    // one app-wide value; every service, untracked
static DB_CREDS: Secret<Json<Credentials>> = Secret::new("DbCreds"); // stored as JSON, T: Deserialize

let key = STRIPE_KEY.value()?;              // Value<String>: derefs to the value
let client = stripe::Client::new(&*key);
let user = &DB_CREDS.value()?.user;
```

- Value types: `String` (default), `Vec<u8>`, `Json<T>`.
- Values refresh in the background. Call `value()` where you use it instead of caching for the process lifetime.
- Unlike TypeScript, a secret with no value is an error even locally: `value()` returns `ErrCode::NotFound` ("secret ... is not configured for this service"). Set it before reading.
- `Value`'s `Debug` prints `Value(<secret>)`, never the value.

Passing a secret to other code: `&impl encore::secrets::AnySecret` accepts any of the three (for `testing::set_secret`). To use the value, pass the `Value` itself.
