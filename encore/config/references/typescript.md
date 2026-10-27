# Config in TypeScript

Package: `@encore.dev/sdk/config`.

## Declare

```typescript
// shop/config.ts
import { config, type Duration, type Int, type Min, type Max } from "@encore.dev/sdk/config";

interface Checkout {
  /** Most items one order may hold. */
  maxItems: number & Int & Min<1> & Max<500>;
  currencies: string[];
  paymentTimeout: Duration;   // "30s" in files; .milliseconds / .seconds in code
  promoCode?: string;         // optional fields may be left out of the files
}

export const checkout = config<Checkout>("checkout");
```

Declare it as a module-level `export const` with a literal name. The type argument must be an interface or type alias.

Scopes: `config<T>()` (service's own, files beside `encore.service.json`), `sharedConfig<T>()` and `globalConfig<T>()` (files beside `encore.app.json`).

Constraints (intersect with the field type): `Min<N>`, `Max<N>`, `Int`, `MinLen<N>`, `MaxLen<N>`, `Pattern<"regex">`, `MinItems<N>`, `MaxItems<N>`.

## Read

```typescript
if (items.length > checkout().maxItems) throw new Error("too many items");
setTimeout(cancelPayment, checkout().paymentTimeout.milliseconds);

const stop = checkout.onChange((value) => log.info("checkout changed", { maxItems: value.maxItems }));
```

Calling the config returns a snapshot that stays the same object until the value changes.

## Secret fields

```typescript
import { config, secret, type Secret } from "@encore.dev/sdk/config";

const stripeKey = secret("StripeKey"); // still declared as a secret

interface Payments { provider: "stripe" | "adyen"; apiKey: Secret<"StripeKey"> }
export const payments = config<Payments>("payments");

const key = payments().apiKey(); // call the field to read the secret
```
