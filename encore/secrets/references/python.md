# Secrets in Python

Import: `from encore.secrets import Secret, SharedSecret, GlobalSecret`.

```python
from encore.secrets import GlobalSecret, Secret, SharedSecret

stripe_key = Secret[str]("StripeKey")               # the declaring service's own
signing_key = SharedSecret[bytes]("SigningKey")     # one app-wide value; readers are tracked
sentry_dsn = GlobalSecret[str]("SentryDSN")         # app-wide, every service, untracked

headers = {"Authorization": f"Bearer {stripe_key.value}"}
```

A shared secret is declared once in the app. Another service (which can't import the declaring service's modules) binds it by name at module scope; its type argument is how it decodes the value:

```python
signing_key = SharedSecret[bytes].named("SigningKey")
```

Only shared secrets bind by name. Declaring the same shared or global name twice in the app is a parse error.

- The type argument is **required on the constructor**: `Secret[str]("X")`. `key: Secret[str] = Secret("X")` raises `TypeError` on `.value`.
- `Secret[str]` decodes UTF-8; `Secret[bytes]` returns raw bytes.
- Reading `.value` raises `KeyError` when no value is set, locally too (TypeScript returns `""` locally instead).
- The value is looked up on first read and cached. A running service keeps the value it started with; `encore run` marks it stale when the value changes.
- Pass `.value` to other code, not the secret object.
