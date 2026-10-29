# Config in Python

Import: `from encore.config import Config, SharedConfig, GlobalConfig`.

## Declare

```python
# orders/settings.py
from dataclasses import dataclass, field
from datetime import timedelta
from typing import Annotated

from encore.config import Config, Max, Min

@dataclass
class Checkout:
    max_items: Annotated[int, Min(1), Max(500)]
    currencies: list[str] = field(default_factory=lambda: ["USD"])
    payment_timeout: timedelta = timedelta(seconds=90)   # "2m" in files

checkout = Config[Checkout]("checkout")                  # type argument required
```

Supported types: a dataclass, `TypedDict` or `Enum`, built from `str`, `int`, `float`, `bool`, `Literal`, `Optional`, `Enum`, `list[...]`, `dict[str, ...]`, nested dataclasses/TypedDicts, `Any`, `timedelta`, and `Secret` fields. Fields with defaults, or `Optional`, may be left out of the files. A union of dataclasses needs a `Literal` tag field.

Constraints, via `Annotated`: `Min(n)`, `Max(n)`, `MinLen(n)`, `MaxLen(n)`, `Pattern(regex)`, `MinItems(n)`, `MaxItems(n)`.

Not supported: Pydantic models, tuples, sets, `bytes`, and dates/times (use `str`). The root type can't be `Optional`.

File keys are the field names as written:

```toml
# orders/checkout.config.toml
max_items = 100
payment_timeout = "2m"
```

## Read

```python
limit = checkout.value.max_items
settings = checkout.value            # each .value read is the newest; keep a local for one consistent snapshot

stop = checkout.on_change(lambda v: log.info("changed", extra={"max_items": v.max_items}))  # runs on a background thread
```

## Secret fields

```python
from encore.secrets import Secret

stripe_key = Secret[str]("StripeKey")   # the secret itself, declared at module scope

@dataclass
class Payments:
    provider_url: str
    api_key: Secret[str] = Secret[str]("StripeKey")   # default names the secret

payments = Config[Payments]("payments")
payments.value.api_key.value
```

The field is always written `Secret[...]`; the config's scope picks the secret's. A `Config` reads the service's own secret, a `SharedConfig` a shared one, a `GlobalConfig` a global one, and the app must declare that secret (`Secret`, `SharedSecret` or `GlobalSecret`) or the parse fails. Secret fields can't be set in files, take no constraints, and can't be `Optional` or inside lists, dicts or unions.
