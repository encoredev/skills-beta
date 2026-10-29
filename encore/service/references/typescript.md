# TypeScript services

## Install the SDK

Pin `@encore.dev/sdk` to the CLI's version. The SDK loads the native runtime that ships with the CLI, so a mismatch fails at startup:

```bash
encore version                       # e.g. 2.0.0-beta.7
bun add @encore.dev/sdk@2.0.0-beta.7 # or npm/pnpm/yarn, matching the service's lockfile
```

`encore service new` pins it for you. `@opentelemetry/api` is a peer dependency, needed for tracing.

## Run strategy

TypeScript has no single startup convention, so the manifest says how to start the service:

```json
"run": { "type": "custom", "cmd": ["bun", "run", "index.ts"] }
"run": { "type": "custom", "cmd": ["node", "dist/main.js"], "build": ["npx", "tsc", "-p", "."] }
"run": { "type": "script", "script": "start", "build": "compile" }
```

`script` runs a `package.json` script through the package manager detected from the lockfile (bun, pnpm, yarn, npm). It defaults to `dev`. Use the runtime the service already uses; Node.js and Bun both work.

## Server

Keep the framework (Express, Hono, Fastify, Next.js, `Bun.serve`). Listen on `ENCORE_LISTEN_ADDR` and drain on shutdown:

```typescript
import express from "express";
import { onShutdown } from "@encore.dev/sdk/shutdown";

const app = express();
// ... routes ...

const addr = process.env.ENCORE_LISTEN_ADDR ?? "127.0.0.1:3000";
const i = addr.lastIndexOf(":");
const server = app.listen(Number(addr.slice(i + 1)), addr.slice(0, i));

onShutdown(() => new Promise<void>((resolve) => server.close(() => resolve())));
```

Bun:

```typescript
const addr = process.env.ENCORE_LISTEN_ADDR ?? "127.0.0.1:3000";
const i = addr.lastIndexOf(":");
Bun.serve({ hostname: addr.slice(0, i), port: Number(addr.slice(i + 1)), fetch: app.fetch });
```

On `SIGTERM`/`SIGINT`, Encore stops taking Pub/Sub messages, waits for handlers, runs `onShutdown` drains, flushes telemetry and exits (0 within the deadline, 1 after it). Pub/Sub subscriptions start when declared, so there's no start call.

## Calling other services

```typescript
import { serviceUrl } from "@encore.dev/sdk/services";

const res = await fetch(new URL("orders/123", serviceUrl("orders"))); // base URL ends in "/"
if (!res.ok) throw new Error(`orders: ${res.status}`);
```

Use a relative path (`"orders/123"`, not `"/orders/123"`) so it resolves under the base URL.

Generated client, when the target declares an `openapi`, `graphql` or `grpc` API and the caller lists it in `"dependencies": ["orders"]`. `encore run` writes it to `encore_gen/clients/orders.ts` (or run `encore gen clients`):

```typescript
import { Client } from "./encore_gen/clients/orders";

const orders = new Client(serviceUrl("orders").href);
const order = await orders.getOrder("123"); // method per operationId; non-2xx throws the client's APIError
```

Options: `{ fetcher }` to replace `fetch`, `{ headers }` for every request. Don't edit `encore_gen/`.

## Metadata

```typescript
import { appMeta, currentRequest } from "@encore.dev/sdk";

appMeta().environment.type;        // "production" | "development" | "ephemeral" | "test"
appMeta().environment.cloud;       // "aws" | "gcp" | "azure" | "encore" | "local"
appMeta().gateways["api"]?.baseUrl;
```

`currentRequest()` describes the Pub/Sub message being handled, or is `undefined`. For HTTP requests, use your framework's request object.

## Deploy builds

The default TS build bundles the service's code with Rolldown and copies the packages it can't bundle into `node_modules`. It keeps `@encore.dev/sdk`, `@prisma/client` and packages with native addons (such as `sharp`) out of the bundle by itself. `"build"` takes:

```json
"build": {
  "entrypoints": ["src/main.ts"],
  "external": ["pino"],
  "inline": ["some-package"],
  "hoist": ["@acme/web"],
  "runtime": "bun"
}
```

- `external`: imported but not bundled; the package is copied in. Use it for a package that reads files beside its code or loads modules by computed names (`__dirname is not defined`, `Cannot find module` when bundled). `encore build dist` on macOS warns that copied native binaries are the host's, not the target's.
- `inline`: bundle a package that was detected as unbundleable but works bundled. It can't also be `external`.
- `hoist`: copied into the build's `node_modules` even though no code imports it.
- `runtime`: `"node"` (default) or `"bun"`, what runs the bundle in the image. `encore run` starts API framework services and Bun templates on Bun, so set `"bun"` to deploy on the runtime you develop on; the build warns when they differ.
- **Files next to the code aren't deployed** (a built SPA in `dist/`, templates, fonts). One way in: make the folder a workspace package (give it a `package.json`, add it to the root `workspaces` and dependencies), `hoist` it, and find it at runtime with `dirname(createRequire(import.meta.url).resolve("@acme/web/package.json"))`. Check the result with `encore build dist --services <name>`.

