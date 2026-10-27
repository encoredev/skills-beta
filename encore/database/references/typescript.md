# Postgres in TypeScript

Package: `@encore.dev/sdk/storage/postgres`, installed at the version `encore version` reports.

## Declare

```typescript
// users/db.ts: its own module, so tests can import it without starting the server
import { Database } from "@encore.dev/sdk/storage/postgres";

export const db = new Database("users", { migrations: "./migrations" });
```

ORM-generated migrations:

```typescript
const db = new Database("users", {
  migrations: { path: "./drizzle", source: "drizzle" }, // "drizzle", "drizzle/v1" or "prisma"
});
```

## Query

Queries are tagged templates. Each `${value}` becomes a bound parameter (`$1`, `$2`, ...), never pasted into the SQL.

```typescript
interface User { id: number; email: string; name: string }

const user = await db.queryRow<User>`SELECT * FROM users WHERE email = ${email}`; // row or null; throws on >1 row
const users = await db.queryAll<User>`SELECT * FROM users ORDER BY name`;       // array

for await (const row of db.query<User>`SELECT * FROM users`) {                  // streamed
  console.log(row.name);
}

const { rowsAffected } = await db.exec`DELETE FROM users WHERE id = ${id}`;     // rowsAffected is a bigint
```

- The type argument describes rows; it does not convert or validate them.
- `BYTEA` arrives as `Uint8Array`. Wrap with `Buffer.from(value)` before calling `Buffer` methods.
- `exec` discards `RETURNING` rows. Use `queryRow` to read them.

Dynamic SQL uses the `raw` variants (`rawQuery`, `rawQueryAll`, `rawQueryRow`, `rawExec`), with parameters passed separately. Only interpolate trusted strings such as column names you choose:

```typescript
const column = sortByName ? "name" : "email"; // never user input
const rows = await db.rawQueryAll<User>(
  `SELECT * FROM users WHERE name ILIKE $1 ORDER BY ${column}`,
  `%${search}%`,
);
```

## Transactions and connections

```typescript
await using tx = await db.begin();
await tx.exec`UPDATE accounts SET balance = balance - ${amount} WHERE id = ${from}`;
await tx.exec`UPDATE accounts SET balance = balance + ${amount} WHERE id = ${to}`;
await tx.commit(); // required: leaving scope ROLLS BACK an uncommitted transaction
```

Without `await using`, call `commit()` or `rollback()` on every path.

Several statements on one connection:

```typescript
await using conn = await db.acquire();
await conn.exec`SET statement_timeout = '5s'`;
```

A streaming `query` holds its connection until the loop ends; leaving the loop early closes it.

## Passing the database to other code

```typescript
import type { Queryable } from "@encore.dev/sdk/storage/postgres";

async function findUser(q: Queryable, id: number) {
  return q.queryRow`SELECT * FROM users WHERE id = ${id}`;
}

await findUser(db.read(), 1);
```

| Handle | Access |
|---|---|
| `db.read()` | Read-only; uses a read replica when the environment has one. Writes fail with SQLSTATE `25006`. |
| `db.read({ primary: true })` | Read-only, always the primary |
| `db.write()` | Read and write |
| `db.admin()` | Read, write and administration |

Calling query methods on the declaration itself counts as write access. Handles, connections and transactions all implement `Queryable`.

Another service's database: `const users = Database.named("users");`

## ORMs and other clients

```typescript
import { drizzle } from "drizzle-orm/node-postgres";
export const orm = drizzle(db.connectionString);
```

`db.read().connectionString` connects read-only. Queries through another client are not in Encore's database spans; add that client's OpenTelemetry instrumentation (see the observability skill).
