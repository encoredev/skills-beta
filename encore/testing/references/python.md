# Testing in Python

`encore test` runs pytest in each Python service's directory, through the package manager detected from its lockfile (`uv run`, `poetry run`, ..., or plain `python3`). A service with no tests passes. **Plain `pytest` fails**: importing `encore` needs the configuration `encore test` supplies.

```bash
encore test
encore test --services orders
encore test uv run pytest -k create_order -x                       # from the service directory
encore test --services orders uv run pytest tests/test_orders.py   # runs in that service's directory
encore test --report-dir out --coverage                            # JUnit + lcov; coverage needs `coverage` in dev deps
```

## Setup

```toml
# pyproject.toml
[dependency-groups]
dev = ["pytest>=8", "pytest-asyncio>=0.24"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]          # tests import service modules the way the service does
asyncio_mode = "auto"       # optional: no @pytest.mark.asyncio needed
```

## Database and storage tests

```python
from uuid import uuid4
import pytest
from resources import db

@pytest.mark.asyncio
async def test_add_user() -> None:
    name = f"user-{uuid4().hex}"
    await db.exec("INSERT INTO users (name) VALUES ($1)", name)
    row = await db.query("SELECT name FROM users WHERE name = $1", name).one()
    assert row == {"name": name}
```

Isolation is **per test process, not per test**. Every test in one pytest run shares the database, buckets, topics and Redis namespace. So:

- Use unique keys and IDs per test (`uuid4()`).
- Clean up in fixtures, so cleanup runs when a test fails.
- Don't assert on table-wide totals; filter on the rows the test created.

## Other services

No other service runs under `encore test`. A module-level `services.client("billing")` still builds, but its requests get `501 Not Implemented`. Mock the client (or the function wrapping the call) in tests.

## Pub/Sub

Subscriptions consume only after `encore.start()`, which nothing calls in tests (unless a test client runs a lifespan that does). Call the handler with a `pubsub.Message` you build (a frozen dataclass):

```python
from datetime import datetime, timezone
from encore import pubsub
from resources import OrderPlaced, record

async def test_record():
    msg = pubsub.Message(
        data=OrderPlaced(order_id=1, customer="t"), raw_data=b"", attributes={},
        id="m1", delivery_attempt=1, publish_time=datetime.now(timezone.utc), ordering_key=None,
    )
    await record(msg)
```

## Django

Use plain pytest and Django's test `Client`, not Django's test runner (it would create its own database). Run Django's migrations yourself in a module-scoped fixture with `call_command("migrate", interactive=False)`. Sync tests call SDK methods through `asgiref.sync.async_to_sync`.
