# Encore.ts v1 to v2

Install `@encore.dev/sdk` at the CLI's version and remove `encore.dev`. With Encore's API framework, also install `@encore.dev/api` at the same version (see `encore-beta-api`).

## API mapping

| v1 (`encore.dev/...`) | v2 (`@encore.dev/sdk/...`) |
|---|---|
| `api()`, `api.raw()` | Same, from `@encore.dev/api` (Encore's API framework), or your framework's routes with an `api` entry |
| `api.streamIn()`, `streamOut()`, `streamInOut()` | Not in the API framework yet: your framework's WebSockets or streaming |
| `api.static()` | A raw catch-all endpoint that serves the files (see `encore-beta-api`) |
| `APIError`, `ErrCode`, `Query<T>`, `Header<T>`, validation types | Same, from `@encore.dev/api`; otherwise your framework's errors and Zod |
| `authHandler`, `Gateway`, `getAuthData` | Middleware on endpoints tagged for auth (see `encore-beta-api`), or your framework's |
| `middleware()` | Same, from `@encore.dev/api`, or your framework's |
| `storage/sqldb`: `SQLDatabase` | `storage/postgres`: `Database` |
| `storage/objects`: `Bucket` | `storage/objects`: `Bucket` (same API) |
| `pubsub`: `Topic`, `Subscription` | `pubsub`: `Topic`, `OrderedTopic`, `Subscription` |
| `storage/cache`: `CacheCluster`, keyspaces | `storage/redis`: `Database` with Redis commands |
| `cron`: `CronJob` with `endpoint` | `cron`: `CronJob` with `path` |
| `config`: `secret()` | `config`: `secret()`, `sharedSecret()`, `globalSecret()` |
| (none) | `config`: `config()`, `sharedConfig()`, `globalConfig()` |
| `metrics`: `Counter`, `CounterGroup`, `Gauge`, `GaugeGroup` | The same plus `UpDownCounter`, `Histogram`, shared/global variants |
| `log` | `log` |
| `appMeta()`, `currentRequest()` from `encore.dev` | Same names from `@encore.dev/sdk` |
| `~encore/clients` | `serviceUrl()` from `services`, or a generated client |

## Changes per API

Databases:

```typescript
// v1
import { SQLDatabase } from "encore.dev/storage/sqldb";
const db = new SQLDatabase("users", { migrations: "./migrations" });
// v2
import { Database } from "@encore.dev/sdk/storage/postgres";
const db = new Database("users", { migrations: "./migrations" });
```

Query methods are the same. `exec` returns `{ rowsAffected: bigint }`; transactions roll back on dispose unless you `commit()` (use `await using`); pass `db.read()`/`db.write()` to helpers; `SQLDatabase.named` becomes `Database.named`.

Pub/Sub handlers get a message wrapper:

```typescript
// v1
handler: async (event) => { await notify(event.orderId); }
// v2
handler: async (message) => { await notify(message.data.orderId); }
```

`orderingAttribute` becomes `OrderedTopic`, with the key passed to `publish`.

Caching:

```typescript
// v1
const cluster = new CacheCluster("sessions", { evictionPolicy: "allkeys-lru" });
const sessions = new StringKeyspace<{ id: string }>(cluster, { keyPattern: "session/:id", defaultExpiry: expireIn(3600 * 1000) });
await sessions.set({ id }, token);
// v2
const sessions = new Database("sessions", { evictionPolicy: "allkeys-lru" });
await sessions.string.setEx(`session/${id}`, token, 3600);
```

Cron:

```typescript
// v1
const _ = new CronJob("cleanup", { every: "1h", endpoint: cleanup });
// v2: and serve POST /internal/cleanup in the framework
export const cleanup = new CronJob("cleanup", { every: "1h", path: "/internal/cleanup" });
```

## Porting an endpoint (Express)

```typescript
// v1
export const get = api(
  { expose: true, method: "GET", path: "/users/:id" },
  async ({ id }: { id: number }): Promise<User> => {
    const user = await db.queryRow<User>`SELECT * FROM users WHERE id = ${id}`;
    if (!user) throw APIError.notFound("user not found");
    return user;
  },
);

// v2: service "users" with { "type": "http", "prefix": "/users", "expose": true }
app.get("/users/:id", async (req, res) => {
  const id = Number(req.params.id);
  if (!Number.isInteger(id)) return res.status(400).json({ error: "invalid id" });
  const user = await db.queryRow<User>`SELECT * FROM users WHERE id = ${id}`;
  if (!user) return res.status(404).json({ error: "user not found" });
  res.json(user);
});
```

## Replacing `~encore/clients`

```typescript
// v1
import { users } from "~encore/clients";
const user = await users.get({ id });
// v2
import { serviceUrl } from "@encore.dev/sdk/services";
const res = await fetch(new URL(`users/${id}`, serviceUrl("users")));
if (!res.ok) throw new Error(`users: ${res.status}`);
const user = (await res.json()) as User;
```

For typed calls, give the target an OpenAPI spec (`{ "type": "openapi", "spec": "./openapi.yaml" }`) and list it in the caller's `dependencies` to get a generated client.

## Pitfalls seen porting a real app

- **Resources passed to helpers:** `helper(db)` fails (ENC0034). Pass `db.read()` or `db.write()` and type the parameter as `Queryable` from `@encore.dev/sdk/storage/postgres`.
- **A shared secret is declared once:** `sharedSecret("X")` in two services fails (ENC0005). Declare it in one shared module and import it. Deployed environments only get app-wide secrets during the beta, so most v1 `secret()` calls become `sharedSecret()`.
- **Config is read inside a service:** calling a config from shared code fails (ENC0035), even a `globalConfig`. Keep the declaration shared and read it in each service.
- **Importing another service's topic:** bind it by name instead, `Topic.named<Event>("events")`, and import only its types (`import type`).
- **`appMeta().apiBaseUrl`** has no replacement: `appMeta().gateways` is empty in services. Put the public URL in config.
- **Route parameter names:** with the API framework, two routes with differently named parameters in the same position fail at startup (502). v1 allowed it.
- **`bodyLimit` on raw endpoints** is ignored: enforce it while reading the body.
- **CORS:** v1's `global_cors` has no automatic equivalent, and the default gateway allows every origin. Add `encore.gateway.json` (see `encore-beta-service`).
- **Tests that called other services** through `~encore/clients` get 501s: serve those endpoints in-process in tests (see `encore-beta-testing`).
- **The UI's v1-generated client** (`encore gen client`) keeps working against API framework services, since paths and error JSON are unchanged, but v2 can't regenerate it yet.

