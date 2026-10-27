# Testing in Python

`encore test` runs `pytest` in each Python service's directory, through the package manager detected from its lockfile (`uv run pytest`, `poetry run pytest`, ..., or `python3 -m pytest`). **Plain `pytest` fails**: importing `encore` needs the configuration `encore test` supplies.

```bash
encore test
encore test --services orders
encore test uv run pytest -k create_order -x                       # from the service directory
encore test --services orders uv run pytest tests/test_orders.py   # runs in that service's directory
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

## Pub/Sub

Nothing calls `encore.run()` or `encore.start()` in tests, so subscriptions don't consume. Call the handler function directly with a `pubsub.Message` you build.

## Django

Use plain pytest and Django's test `Client`, not Django's test runner (it would create its own database). Run Django's migrations yourself in a module-scoped fixture with `call_command("migrate", interactive=False)`. Sync tests call SDK methods through `asgiref.sync.async_to_sync`.
