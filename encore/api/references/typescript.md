# Encore's API framework in TypeScript

Package: `@encore.dev/api`, with `@encore.dev/sdk` and `@opentelemetry/api` as peers. Pin both Encore packages to the CLI's version:

```bash
bun add @encore.dev/api@2.0.0-beta.5 @encore.dev/sdk@2.0.0-beta.5 @opentelemetry/api
```

Encore generates the service's entry (`node_modules/.encore/<service>/entry.ts`) and runs it with Bun. The entry imports the service's `index.ts` and **only the modules that declare endpoints**. A subscription or cron job declared in a module nothing imports never starts: declare it in an endpoint module or a module one imports.

## Endpoints

```typescript
import { api, APIError, type Header, type Query, type Min, type Max, type MaxLen } from "@encore.dev/api";

interface ListParams {
  id: number;                                   // the path's :id
  limit?: Query<number> & Min<1> & Max<100>;
  locale?: Header<"Accept-Language">;
}

export const get = api(
  { expose: true, method: "GET", path: "/users/:id", tags: ["auth"] },
  async (p: ListParams): Promise<User> => {
    const user = await find(p.id);
    if (!user) throw APIError.notFound("no such user");
    return user;
  },
);

// Private: other services and cron jobs can call it, the gateway can't.
export const prune = api({ method: "POST", path: "/users/prune" }, async (): Promise<void> => { /* ... */ });
```

- Options must be literals: `method` (one, a list or `"*"`), `path` (`:id` or `{id}`, last segment `{*rest}`), `expose`, `tags`, `sensitive` (kept out of traces), `bodyLimit` (bytes, 2 MiB by default, `null` for none).
- **`bodyLimit` is missing from the `APIOptions` type in beta.5** though Encore reads it. Add once, e.g. `lib/api-options.d.ts`:
  ```typescript
  import "@encore.dev/api";
  declare module "@encore.dev/api" { interface APIOptions { bodyLimit?: number | null } }
  ```
- Request fields that aren't path, `Query`, `Header` or `Cookie` come from the JSON body; for GET, HEAD and DELETE they're query parameters. A request that doesn't decode or validate answers 400 `invalid_argument` naming the field. Validation types: `Min`, `Max`, `MinLen`, `MaxLen`, `IsEmail`, `IsURL`, `StartsWith`, `EndsWith`, `MatchesRegexp`.
- Response fields typed `Header<>`, `Cookie<>` or `HttpStatus` become headers, cookies and the status. A thrown `APIError` answers with its code's status; any other error answers 500 `internal` and is logged.

## Raw endpoints

```typescript
import { api, currentRequest } from "@encore.dev/api";

export const upload = api.raw(
  { expose: true, tags: ["auth"], method: "PUT", path: "/files/:id" },
  async (req, resp) => {
    const meta = currentRequest();
    const id = meta?.pathParams.id;
    // ... read req, write resp
    resp.end();
  },
);
```

- **Raw endpoints have no body limit**, whatever `bodyLimit` says. Count bytes while reading and reject past your limit.
- The handler must return `void`/`Promise<void>`: write `resp.end(); return;`, not `return resp.end()`, or it doesn't type-check.
- `currentRequest()` from `@encore.dev/api` describes the API call (`pathParams`, `headers`, `api.tags`, `middlewareData`). The same name in `@encore.dev/sdk` describes a Pub/Sub message instead.

## Serving files or a single-page app

There's no `api.static`. Serve files from a raw catch-all, plus `/`, which the wildcard doesn't match. Other services' more specific routes still win.

```typescript
export const assets = api.raw({ expose: true, method: ["GET", "HEAD"], path: "/{*path}" }, serveFile);
export const home = api.raw({ expose: true, method: ["GET", "HEAD"], path: "/" }, serveFile);
```

`serveFile` maps the URL to a file under the build folder (guard against `..`), sets the content type, and answers `index.html` with 200 for app routes. The deploy build bundles code only, so files next to the service aren't deployed: see "Deploy builds" in `encore-beta-service`'s TypeScript reference.

## Middleware and auth

There's no auth handler yet. Tag endpoints and authenticate in middleware, which the service's `index.ts` installs:

```typescript
// lib/auth.ts, shared by the services
import { APIError, currentRequest, middleware } from "@encore.dev/api";

export interface AuthData { userID: string }

export function authMiddleware(identify: (headers: Record<string, string | string[]>) => Promise<AuthData | null>) {
  return middleware(async (req, next) => {
    const tags = req.requestMeta.api.tags;
    const required = tags.includes("auth");
    if (!required && !tags.includes("optional-auth")) return next(req);
    const data = await identify(req.requestMeta.headers).catch((err) => { if (required) throw err; return null; });
    if (!data && required) throw APIError.unauthenticated("not signed in");
    if (data) req.data.auth = data;          // read back via currentRequest().middlewareData
    return next(req);
  });
}

export const getAuthData = () => (currentRequest()?.middlewareData?.auth as AuthData | undefined) ?? null;
```

```typescript
// users/index.ts
import { Service } from "@encore.dev/api";
import { authMiddleware } from "../lib/auth";
export default new Service({ middlewares: [authMiddleware(identify)] });
```

- `middleware({ target: { expose, isRaw, tags } }, fn)` limits which endpoints it runs for. A middleware can answer by throwing `APIError` or returning `new HandlerResponse(payload)`; after `next()` it can set `resp.header` and `resp.status`.
- If one service owns sessions (say `auth`), others can't share its code's database or library state: give it a private endpoint that takes the cookie or `Authorization` header and returns the caller (mark it `sensitive: true`), and have the other services' `identify` call it. That's one extra hop per authenticated request.
- v1 carried auth across service calls; v2 doesn't. To call another service as the same user, pass on the request's `Cookie`/`Authorization` headers and let it check them again.

## Calling other services

No typed clients are generated between `encore-api` services yet (`dependencies` + `encore gen clients` writes nothing). Call the base URL and reuse the target's types with a type-only import, which adds no runtime dependency:

```typescript
import { APIError, ErrCode } from "@encore.dev/api";
import { serviceUrl } from "@encore.dev/sdk/services";
import type * as users from "../users/users";

type Req<F> = F extends (p: infer P) => Promise<unknown> ? P : never;
type Resp<F> = F extends (...a: never[]) => Promise<infer R> ? R : never;

export async function getUser(p: Req<typeof users.get>): Promise<Resp<typeof users.get>> {
  const resp = await fetch(new URL(`users/${p.id}`, serviceUrl("users")));  // relative path
  const body = await resp.json();
  if (!resp.ok) throw new APIError(body.code ?? ErrCode.Internal, body.message ?? `users: ${resp.status}`);
  return body;
}
```

Private endpoints are reachable this way. Responses arrive as JSON, so `Date` fields are strings.

## Testing

An endpoint value is callable as its handler: `await get({ id: 1 })`. Under `encore test` no service runs, so for code that calls another service, have the call helper invoke the target endpoint in-process in tests (the test process can reach every service's database). Register those endpoints lazily, on first call, or a test file's `vi.mock()` won't apply to modules a setup file already imported. Middleware doesn't run on direct calls: set the caller in tests some other way (e.g. an `AsyncLocalStorage` that `getAuthData()` checks first). See `encore-beta-testing` for running the suite.
