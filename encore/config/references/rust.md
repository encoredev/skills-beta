# Config in Rust

Module: `encore::config`.

## Declare

```rust
use encore::config::{Config, ConfigType, Duration};
use serde::Deserialize;

#[derive(Debug, Deserialize, ConfigType)]
pub struct Checkout {
    /// Most items one order may hold.
    #[config(min = 1, max = 500)]
    pub max_items: u32,
    pub currencies: Vec<String>,
    pub payment_timeout: Duration,  // "30s" in files; derefs to std::time::Duration
    #[serde(default)]
    pub promo_code: Option<String>, // fields with a default may be left out of the files
}

pub static CHECKOUT: Config<Checkout> = Config::new("checkout");
```

Derive both `Deserialize` and `ConfigType`. The type is the schema: Encore reads its fields, serde attributes, `#[config(..)]` constraints and doc comments, and checks every value before the service sees it.

Scopes: `Config<T>` (service's own, files beside `encore.service.json`), `SharedConfig<T>` and `GlobalConfig<T>` (files beside `encore.app.json`).

Constraints (`#[config(..)]` on a field): `min = N`, `max = N`, `min_len = N`, `max_len = N`, `pattern = "regex"`, `min_items = N`, `max_items = N`.

Not supported in a config type: tuples, maps with non-string keys, untagged enums.

## Read

```rust
let checkout = CHECKOUT.get();               // Value<Checkout>: a snapshot, never fails
if items.len() > checkout.max_items as usize { /* too many */ }
tokio::time::sleep(*checkout.payment_timeout).await;

CHECKOUT.on_change(|value| log::info!(max_items = value.max_items; "checkout changed"));
```

Hold one snapshot for a request so its values stay consistent; call `get()` again for the next. `on_change` runs on the runtime's threads: keep it quick.

## Secret fields

```rust
use encore::secrets::Secret;

static STRIPE_KEY: Secret = Secret::new("StripeKey"); // still declared as a secret

#[derive(Deserialize, ConfigType)]
pub struct Payments {
    pub provider: String,
    #[config(secret = "StripeKey")]
    pub api_key: Secret,
}
pub static PAYMENTS: Config<Payments> = Config::new("payments");

let key = PAYMENTS.get().api_key.value()?;
```

No config file sets a secret field. The secret has the config's scope: `Config` reads the service's own secret, `SharedConfig`/`GlobalConfig` an app-wide one.
