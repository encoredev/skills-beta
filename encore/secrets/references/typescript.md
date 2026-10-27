# Secrets in TypeScript

Package: `@encore.dev/sdk/config`.

```typescript
import { secret, sharedSecret, globalSecret } from "@encore.dev/sdk/config";

const stripeKey = secret("StripeKey");         // the declaring service's own
const signingKey = sharedSecret("SigningKey"); // one app-wide value; readers are tracked
const sentryDsn = globalSecret("SentryDSN");   // one app-wide value; every service, untracked

const stripe = new Stripe(stripeKey());        // a secret is a function: call it to read
```

- Values refresh in the background. Read where you use it instead of caching for the process lifetime.
- Locally, a secret with no value reads as `""`, so the app still starts. In a deployed environment, reading a missing secret throws.
- `${stripeKey}` stringifies to `Secret<StripeKey>(*********)`, never the value.

Passing a secret to other code:

```typescript
import type { Secret } from "@encore.dev/sdk/config";
function stripeClient(key: Secret<"StripeKey">) { return new Stripe(key()); }
```

`AnySecret` accepts any secret.
