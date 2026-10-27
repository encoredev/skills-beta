# Config in Go

Package: `sdk.encore.dev/config`.

## Declare

```go
// shop/config.go
import "sdk.encore.dev/config"

type Checkout struct {
	// Most items one order may hold.
	MaxItems       int             `json:"max_items" config:"min=1,max=500"`
	Currencies     []string        `json:"currencies"`
	PaymentTimeout config.Duration `json:"payment_timeout"` // "30s" in files; embeds time.Duration
	PromoCode      *string         `json:"promo_code"`      // optional: pointer, omitempty or config:"optional"
}

var checkout = config.New[Checkout]("checkout")
```

```toml
# shop/checkout.config.toml
max_items = 20
currencies = ["usd", "eur"]
payment_timeout = "30s"
```

Declare it as a package-level variable with a literal name. Every field is required unless optional. A key is the field's `json` name (or its Go name), matched exactly, case included.

Scopes: `config.New[T]()` (service's own, files beside `encore.service.json`), `config.NewShared[T]()` and `config.NewGlobal[T]()` (files beside `encore.app.json`).

Constraints in the `config` tag: `min=`, `max=`, `min_len=`, `max_len=`, `min_items=`, `max_items=`, `pattern=` (must come last).

## Read

```go
if len(items) > checkout.Get().MaxItems {
	return errors.New("too many items")
}
time.AfterFunc(checkout.Get().PaymentTimeout.Duration, cancelPayment)

checkout.OnChange(func(c Checkout) { slog.Info("checkout changed", "max_items", c.MaxItems) })
```

`Get()` never fails: the value is checked against the type before the service starts. Each value is decoded once; treat it as read-only (slices and maps are shared). `OnChange` runs `f` on its own goroutine, coalescing fast changes.

## Secret fields

```go
var stripeKey = secrets.New[string]("StripeKey") // still declared as a secret

type Payments struct {
	Provider string                  `json:"provider"`
	APIKey   *secrets.Secret[string] `json:"api_key" config:"secret=StripeKey"`
}

var payments = config.New[Payments]("payments")

key := payments.Get().APIKey.StringValue() // or .Value()
```

The secret is of the config's scope and never set from the config files.
