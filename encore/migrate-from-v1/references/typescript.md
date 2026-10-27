# Encore.ts v1 to v2

Install `@encore.dev/sdk` at the CLI's version and remove `encore.dev`.

## API mapping

| v1 (`encore.dev/...`) | v2 (`@encore.dev/sdk/...`) |
|---|---|
| `api()`, `api.raw()`, `api.streamIn()`, ... | Your framework's routes, declared to Encore with an `api` entry |
| `APIError`, `ErrCode` | Your framework's error handling |
| `Query<T>`, `Header<T>`, validation types | Framework validation or Zod |
| `authHandler`, `Gateway`, `getAuthData` | Framework auth middleware |
| `middleware()` | Framework middleware |
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
