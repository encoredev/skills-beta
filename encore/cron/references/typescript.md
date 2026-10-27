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

v1 took `endpoint: someApi`; v2 takes `path`.
