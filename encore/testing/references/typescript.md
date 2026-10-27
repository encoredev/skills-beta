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

**Known issue (2.0.0-beta.5):** `encore test` and `encore exec` don't tell the TypeScript SDK where its native runtime is, so tests fail with `could not find the Encore runtime library ('encore-runtime.node')`. `encore run` is unaffected. Set the path yourself:

```bash
export ENCORE_RUNTIME_LIB="$(brew --prefix encore-beta)/libexec/runtimes/js/encore-runtime.node"
encore test
```

Or set it in `encore.service.json`: `"test": { "type": "script", "script": "test:integration" }` or `{ "type": "custom", "cmd": [...] }`.

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

## Pub/Sub

Export the handler function and call it with a message you build:

```typescript
export async function handleOrderPlaced(message: Message<OrderPlaced>) { /* ... */ }
new Subscription(orderPlaced, "send-confirmation", { handler: handleOrderPlaced });
```
