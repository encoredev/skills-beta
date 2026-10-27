# Python services

## Install the SDK

The distribution is `encore-dev`, imported as `encore`. Pin it to the CLI's version (CLI `2.0.0-beta.5` is `encore-dev==2.0.0b5`):

```toml
[project]
requires-python = ">=3.14"           # what the official templates use; the SDK itself needs >=3.9
dependencies = ["encore-dev==2.0.0b5", "fastapi>=0.143"]
```

The package is pure Python. Its native runtime ships with the CLI, and **importing `encore` reads configuration from Encore**, so start code through `encore run`, `encore test` or `encore exec`, never `python main.py` alone.

## Run strategy

The default (`"type": "python"`, or no `run`) runs `main.py` (or `app.py`) through the package manager found from the lockfile in the service directory:

| Lockfile | Run | Test |
|---|---|---|
| `uv.lock` | `uv run main.py` | `uv run pytest` |
| `poetry.lock` | `poetry run python main.py` | `poetry run pytest` |
| `Pipfile.lock`/`Pipfile` | `pipenv run python main.py` | `pipenv run pytest` |
| `pdm.lock` | `pdm run python main.py` | `pdm run pytest` |
| none | `python3 main.py` | `python3 -m pytest` |

Anything else: `"run": {"type": "custom", "cmd": ["uv", "run", "main.py"]}` (what the templates use). It runs from the service directory, not through a shell. `run.command` isn't a field.

## Serve with `encore.run(app)` (recommended)

```python
# main.py
import encore
from fastapi import FastAPI, HTTPException

from resources import db   # import every module that declares resources or subscriptions

app = FastAPI()

@app.get("/orders/{order_id}")
async def get_order(order_id: int) -> dict:
    row = await db.query("SELECT id, total FROM orders WHERE id = $1", order_id).first()
    if row is None:
        raise HTTPException(status_code=404, detail="order not found")
    return row

if __name__ == "__main__":
    encore.run(app)
```

`encore.run(app)` serves any ASGI app on Encore's HTTP server at `ENCORE_LISTEN_ADDR`, runs the app's lifespan, starts Pub/Sub subscriptions, records a server span per request (and installs Encore's tracer provider as the global), and shuts down gracefully on SIGTERM/SIGINT. No uvicorn, no `encore.start()`, no shutdown code.

- WSGI apps (Flask, Django): wrap them, e.g. `encore.run(WSGIMiddleware(app))` with `from a2wsgi import WSGIMiddleware`.
- A service with no HTTP API (only subscriptions): `encore.run()`.
- Options: `workers` (more than 1 needs free-threaded Python), `max_concurrency`, `read_timeout`, `tracing`, `server_spans`.
- Don't fork after importing `encore` (no gunicorn/uvicorn worker processes).

## Or keep your own server

Then start subscriptions and shut down yourself:

```python
import os
from contextlib import asynccontextmanager

import encore
import uvicorn
from encore import shutdown

@asynccontextmanager
async def lifespan(app: FastAPI):
    await encore.start()          # REQUIRED for subscriptions to receive anything
    try:
        yield
    finally:
        shutdown.begin()
        await shutdown.completed()

app = FastAPI(lifespan=lifespan)

if __name__ == "__main__":
    host, _, port = os.environ.get("ENCORE_LISTEN_ADDR", "0.0.0.0:8000").rpartition(":")
    uvicorn.run(app, host=host, port=int(port))
```

Turn off the framework's reloader; `encore run` restarts on change. Background workers using resources: `shutdown.begin(cooperative=True)`, finish the work, `shutdown.mark_application_drained()`, then `await shutdown.completed()`.

## Routes include the prefix

The service receives the full request path, with its `api` prefix. A service declaring `"prefix": "/notes"` serves `/notes/...` routes, e.g. `APIRouter(prefix="/notes")`.

## Sync code

Django views and Flask call the async SDK through `asgiref.sync.async_to_sync`:

```python
data = async_to_sync(uploads.download)(name)
```

## Calling other services

```python
import httpx
from encore.services import base_url

async with httpx.AsyncClient() as client:
    r = await client.get(f"{base_url('orders')}/orders/{order_id}")   # no trailing slash
    r.raise_for_status()
```

`base_url` takes a string literal, must be called directly (not stored or passed around), from code inside a service. Unknown names raise `ValueError`.

## Metadata

```python
import encore
meta = encore.app_meta()           # also encore.env.app_meta()
meta.environment.type              # "production" | "development" | "ephemeral" | "test"
meta.environment.cloud             # "aws" | "gcp" | "azure" | "encore" | "local"
meta.gateways.get("api")           # GatewayMeta(name, base_url) or None
```
