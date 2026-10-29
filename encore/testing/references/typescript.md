# Testing in TypeScript

`encore test` runs each service's `test` strategy. The default runs the `test` script in `package.json` through the package manager detected from the lockfile. Any runner works: Vitest, Jest, `node --test`, `bun test`.

```json
{ "scripts": { "test": "vitest run" } }
```

```bash
encore test                                   # every service
encore test --services users                  # one service
encore test -- vitest run users.test.ts       # a command of your own
```

To run something else by default, set it in `encore.service.json`: `"test": { "type": "script", "script": "test:integration" }` or `{ "type": "custom", "cmd": [...] }`. With several TS services sharing one `package.json`, run the suite once with your own command (`encore test -- vitest run`) rather than once per service.

## Vitest: one process per run

Each test **process** gets its own copy of every database it touches, dropped when the process exits. Vitest's default `forks` pool starts a process per test file, so a suite of 40 files and 4 databases creates and drops about 160 databases a run. Every `DROP DATABASE` forces a checkpoint; the test Postgres backs up for minutes and tests time out. Run all files in one process with the `threads` pool, which still isolates each file's modules:

```typescript
// vitest.config.ts
export default defineConfig({ test: { pool: "threads" } });
```

In one real suite this took a run from 4 to 20 minutes with dozens of timeouts to 60 seconds, all passing. `CREATE DATABASE` or `DROP DATABASE` processes that linger after a run (`ps aux | grep DATABASE`) are this backlog.

## Database tests

```typescript
// users/users.test.ts
import { describe, expect, it } from "vitest";
import { db } from "./db";

describe("users", () => {
  it("inserts and reads back a user", async () => {
    await db.exec`INSERT INTO users (name) VALUES (${"Ada"})`;
    const row = await db.queryRow`SELECT name FROM users WHERE name = ${"Ada"}`;
    expect(row?.name).toBe("Ada");
  });
});
```

The database is shared by every test file in the run. If files run in parallel against the same tables, use unique values per test, clean up, or run files serially.

## HTTP route tests

Export the framework app from a module that doesn't call `listen`, and test it with `supertest` or the framework's helpers (`app.request()` in Hono, `app.inject()` in Fastify):

```typescript
// app.ts exports `app`; index.ts imports it and listens
import request from "supertest";
import { app } from "./app";

it("404s for a missing user", async () => {
  await request(app).get("/users/999999").expect(404);
});
```

## Other services

`serviceUrl("orders")` returns a URL, but requests get `501 Not Implemented`. Mock them, for example with a `fetcher` for a generated client:

```typescript
const orders = new Client(serviceUrl("orders").href, {
  fetcher: (async () => new Response(JSON.stringify({ id: "1" }), { status: 200 })) as typeof fetch,
});
```

**Calling another service's endpoints in-process.** The test process can reach every service's database, so a call helper can invoke the target's endpoint function directly under test instead of over HTTP (check `appMeta().environment.type === "test"`). Register the endpoints lazily, on the first call (`() => import("../orders/orders")`): modules a setup file imports eagerly are loaded before a test file's `vi.mock()` applies, so the mocks silently don't take effect. JSON round-trip the result so tests see what HTTP would deliver (`Date` fields as strings).

## Pub/Sub

Export the handler function and call it with a message you build:

```typescript
export async function handleOrderPlaced(message: Message<OrderPlaced>) { /* ... */ }
new Subscription(orderPlaced, "send-confirmation", { handler: handleOrderPlaced });
```

Topics are real under `encore test`, so publishing works as in `encore run`.
