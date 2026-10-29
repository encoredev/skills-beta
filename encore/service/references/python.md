# Python services

## Install the SDK

The distribution is `encore-dev`, imported as `encore`. Pin it to the CLI's version (CLI `2.0.0-beta.7` is `encore-dev==2.0.0b7`):

```toml
[project]
requires-python = ">=3.14"           # what the official templates use; the SDK itself needs >=3.9
dependencies = ["encore-dev[httpx]==2.0.0b7", "fastapi>=0.143"]
```

The package is pure Python. Its native runtime ships with the CLI, and **importing `encore` reads configuration from Encore**, so start code through `encore run`, `encore test` or `encore exec`, never `python main.py` alone.

Extras add the service-call adapters: `httpx`, `requests`, `aiohttp`, `niquests` (`encore-dev[httpx]`).

Deployed images use `python:<major.minor>-slim`, with the version from `.python-version`, else from `requires-python`. With neither, the build fails unless `encore.service.json` sets `"docker": {"base_image": ...}`.

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

`encore.run(app)` serves any ASGI app on Encore's HTTP server (HTTP/1.1 and h2c) at `ENCORE_LISTEN_ADDR`, runs the app's lifespan, starts Pub/Sub subscriptions, gives each request its Request Context and a server span named after the matched route (and installs Encore's tracer provider as the global), and shuts down gracefully on SIGTERM/SIGINT. No uvicorn, no `encore.start()`, no shutdown code.

- WSGI apps (Flask, Django): wrap them, e.g. `encore.run(WSGIMiddleware(app))` with `from a2wsgi import WSGIMiddleware`.
- A service with no HTTP API (only subscriptions): `encore.run()`.
- Options: `workers` (more than 1 needs free-threaded Python), `max_concurrency` (default no limit), `read_timeout` (seconds, default 30), `tracing`, `server_spans` (False leaves the request span to FastAPI), `host`/`port` (default: Encore's address).
- WebSockets aren't supported by `encore.run` yet; keep your own server for them.
- Don't fork after importing `encore` (no gunicorn/uvicorn worker processes).

## Or keep your own server

Then add Encore's middleware (Request Context and a server span per request) and start the subscriptions yourself:

```python
import os
from contextlib import asynccontextmanager

import encore
import uvicorn
from encore.server import asgi

@asynccontextmanager
async def lifespan(app: FastAPI):
    await encore.start()          # REQUIRED for subscriptions to receive anything
    yield

app = FastAPI(lifespan=lifespan)
app.add_middleware(asgi.Middleware)   # WSGI: app = encore.server.wsgi.Middleware(app)

if __name__ == "__main__":
    host, _, port = os.environ.get("ENCORE_LISTEN_ADDR", "0.0.0.0:8000").rpartition(":")
    uvicorn.run(app, host=host, port=int(port))
```

Turn off the framework's reloader; `encore run` restarts on change. Encore cleans up its resources when the interpreter exits. To drain background work that uses resources first: `shutdown.begin(cooperative=True)` (`from encore import shutdown`), finish the work, `shutdown.mark_application_drained()`, then `await shutdown.completed()`.

## Routes include the prefix

The service receives the full request path, with its `api` prefix. A service declaring `"prefix": "/notes"` serves `/notes/...` routes, e.g. `APIRouter(prefix="/notes")`.

## Sync code

Django views and Flask call the async SDK through `asgiref.sync.async_to_sync`:

```python
data = async_to_sync(uploads.download)(name)
```

## Calling other services

Use the adapter for your HTTP client. Its `client(service)` resolves the base URL and sends the Request Context (only to the app's services):

```python
from encore.services import httpx as services

orders = services.client("orders")                      # httpx.AsyncClient; relative URLs are the service's

async def get_order(order_id: int) -> dict:
    r = await orders.get(f"/orders/{order_id}")
    r.raise_for_status()
    return r.json()
```

- Adapters: `encore.services.httpx` (`client`, `sync_client`), `.requests` (`client`), `.aiohttp` (`client`; create it inside a coroutine, paths without the leading slash), `.niquests` (`client`, `sync_client`). Each needs its extra.
- `base_url("orders")` (from `encore.services` or an adapter) gives the URL alone, without a trailing slash. A plain client then doesn't send the Request Context.
- The service name is a string literal, and `client`/`base_url` are called directly (not stored or passed around), from code inside a service. Unknown names raise `ValueError`.
- A generated client (`encore_gen/clients/<service>.py`, for an `openapi` or `graphql` target listed in `dependencies`) is synchronous httpx: `Client(base_url("users"), http_client=services.sync_client("users"))`.

## Request Context and gateway Middleware

Request Context fields are typed values that travel with a request to every service it reaches. No client can set them.

```python
from dataclasses import dataclass
from encore import reqctx

@dataclass
class User:
    id: str

current_user = reqctx.Field[User]("user")            # declared once in the app; kebab-case name
# another service: current_user = reqctx.Field[User].named("user")

user = current_user.get()                             # None when not set
with current_user.set(User(id="system")):             # for the calls made in the block
    await billing.get("/invoices")
```

Options: `pubsub=True` carries the field on published messages into their handlers; `trace=True` records it on spans (never for secrets). Values encode to at most 4 KiB (1 KiB with `pubsub`).

A gateway with code is a service whose `encore.service.json` has a `gateway` section instead of `api`, e.g. `{"name": "edge", "lang": "python", "gateway": {"default": true}}`. Its `main.py` serves Middleware that authenticate public requests and set fields:

```python
import asyncio
import encore
from encore import gateway
from encore.errs import ErrCode
from encore.gateway import match

class Authenticate(gateway.Middleware):
    match = match.path_prefix("/api")                 # optional; every request by default

    async def on_request(self, req: gateway.Request) -> None:
        user = await verify(req.header("authorization"))
        if user is None:
            raise gateway.reject(ErrCode.UNAUTHENTICATED, "missing token")
        req.set(current_user, user)

async def main() -> None:
    await encore.start()
    await gateway.serve(Authenticate())

asyncio.run(main())
```

Other hooks: `on_request_body`, `on_response`, `on_response_body`, `on_complete`. Settings are class attributes: `name`, `match`, `timeout` (5s), `on_error` (`ErrorPolicy.FAIL_CLOSED`). Middleware only see public requests, not calls between services.

## Metadata

```python
import encore
meta = encore.app_meta()           # also encore.env.app_meta()
meta.environment.type              # "production" | "development" | "ephemeral" | "test"
meta.environment.cloud             # "aws" | "gcp" | "azure" | "encore" | "local"
meta.gateways.get("api")           # GatewayMeta(name, base_url) or None; empty under encore test/exec
```
