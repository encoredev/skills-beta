# Cron jobs in TypeScript

Package: `@encore.dev/sdk/cron`.

```typescript
import { CronJob } from "@encore.dev/sdk/cron";

export const cleanup = new CronJob("cleanup-expired-sessions", {
  title: "Delete expired sessions",   // optional
  path: "/internal/cleanup",
  every: "1h",                        // or schedule: "0 4 * * *" (UTC). One of the two.
});
```

Serve the path as a `POST` route with your framework:

```typescript
app.post("/internal/cleanup", async (_req, res) => {
  const { rowsAffected } = await db.exec`DELETE FROM sessions WHERE expires_at < now()`;
  res.json({ deleted: Number(rowsAffected) });
});
```

On Encore's API framework (`"type": "encore-api"`), serve it with a private endpoint at the same path, in a module the service's entry imports:

```typescript
import { api } from "@encore.dev/api";

export const cleanupExpired = api({ method: "POST", path: "/internal/cleanup" }, async (): Promise<void> => {
  await db.exec`DELETE FROM sessions WHERE expires_at < now()`;
});
```

v1 took `endpoint: someApi`; v2 takes `path`.
