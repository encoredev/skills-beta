# Redis in TypeScript

Package: `@encore.dev/sdk/storage/redis`. Replaces v1's `CacheCluster` and keyspaces.

## Declare

```typescript
import { Database } from "@encore.dev/sdk/storage/redis";

export const sessions = new Database("sessions");                                  // a cache: may evict
export const ledger = new Database("ledger", { persistent: true });               // never evicted
export const pageCache = new Database("page-cache", { evictionPolicy: "allkeys-lfu" }); // default "allkeys-lru"
```

Name: kebab-case literal. A persistent database takes no `evictionPolicy` (parse error). Another service's database: `Database.named("sessions")`.

## Commands

Commands are grouped by data type and named after the Redis command:

```typescript
await sessions.string.set(`session:${id}`, JSON.stringify(session));
await sessions.string.setEx(`otp:${userId}`, code, 300);                          // seconds
await sessions.string.setOptions(key, value, { condition: "nx", expiry: { ex: 60 } });
await sessions.string.incr(`visits:${userId}`, 1);
await sessions.hash.hset(`user:${userId}`, "name", name);
await sessions.list.lpush("jobs", jobId);
await sessions.set.sadd(`tags:${postId}`, "typescript");
await sessions.sortedSet.zadd("leaderboard", userId, score);
await sessions.generic.expire(`session:${id}`, 3600);
await sessions.generic.del(`session:${id}`);
```

Groups: `string`, `hash`, `list`, `set`, `sortedSet`, `stream`, `geo`, `bitmap`, `pubsub`, `generic`.

**Call a command in the same expression as its group** (`sessions.string.get(...)`). Storing a group (`const s = sessions.string`) is a parse error.

## Reading replies

Replies with a fixed type come back directly (`exists` → boolean, `strlen` → number). Value replies are a `ReplyPromise`; pick the type:

```typescript
const name = await sessions.string.get(`name:${userId}`).text();       // string | null
const visits = await sessions.string.get(`visits:${userId}`).number(); // number | null
const user = await sessions.hash.hgetall(`user:${userId}`).textMap();  // Record<string, string>
const top = await sessions.sortedSet.zrangeWithscores("leaderboard", 0, 9).scoredMembers();
```

Other readers: `bytes()`, `bigint()`, `bool()`, `texts()`, `numbers()`, `optionalTexts()` (for `mget`), `value()`.

## Pipelines, transactions, watch, scripts

```typescript
const pipe = sessions.pipeline();            // one round trip, not atomic
pipe.string.get("name:1");
pipe.string.get("name:2");
const [a, b] = await pipe.exec();

const tx = sessions.transaction();           // MULTI/EXEC; failing command => exec throws BatchError
tx.string.incr("hits", 1);
await tx.exec();

const left = await sessions.watch(["balance"], async (conn, tx) => { // optimistic, retried on conflict
  const balance = (await conn.string.get("balance").number()) ?? 0;
  tx.string.set("balance", balance - 10);
  return balance - 10;
});

import { Script } from "@encore.dev/sdk/storage/redis";
const release = new Script(`if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) end return 0`);
const ok = await locks.script(release).key(`lock:${job}`).arg(token).invoke().bool();
```

Pass every key a script touches with `.key()`: only those get the database's key prefix.

## Passing the database to other code

```typescript
import type { ReadOnlyDatabaseRef } from "@encore.dev/sdk/storage/redis";
function nameOf(db: ReadOnlyDatabaseRef, user: string) { return db.string.get(`name:${user}`).text(); }
await nameOf(sessions.read(), userId);
```

Handles: `read()` (may use a replica), `readPrimary()`, `write()`.

## Other clients

`sessions.connectionString` works with other Redis clients, but bypasses the key prefix, so keys don't match the SDK's.
