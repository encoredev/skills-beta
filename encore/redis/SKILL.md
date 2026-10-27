---
name: encore-beta-redis
description: Use Redis in an Encore v2 (beta) app in TypeScript, Python, Go or Rust, for caching, counters, rate limits, sessions, queues and leaderboards. Covers declaring a Redis database, command groups, reading typed replies, TTLs, pipelines and transactions where supported, read/write handles, and other Redis clients.
when_to_use: >-
  User is in an Encore v2 app and wants to cache something, store sessions or OTPs with a TTL, count or rate-limit, build a leaderboard, use Redis lists or streams, or replace v1 cache keyspaces. Trigger phrases: "Redis", "cache", "TTL", "expire", "rate limit", "session store", "leaderboard", "counter", "key-value", "CacheCluster", "keyspace". SKIP for durable messaging (use `encore-beta-pubsub`) and v1 apps.
---

# Redis in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

v2 replaces v1's cache clusters and typed keyspaces with a Redis database and direct Redis commands. Encore provisions local Redis for `encore run`, a namespace per test run, and ElastiCache or Memorystore when deployed.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`). Support varies: TypeScript and Go have pipelines, transactions, `watch` and Lua scripts; Python has none of those yet.

## Rules for every language

- Declare at module/package scope. Name: a kebab-case string literal, unique in the app. Never rename a deployed database.
- Without persistence, a database is a cache: data may be evicted or lost. Don't keep the only copy of anything there.
- Commands are grouped by data type (string, hash, list, set, sorted set, stream, geo, bitmap, pubsub, generic) and named after the Redis command. **Call a command on its group in one expression** (`cache.string.get(...)`). Storing a group in a variable is a parse error, because Encore reads each call to grant access.
- Encore prefixes every key per database (and per test run). You always read and write unprefixed keys. Clients connected through the raw connection string bypass the prefix and won't see the same keys.
- Values come back as bytes or a reply you decode explicitly; pick the type when reading. A missing key reads as null/None.
- Hand the database to other code as a handle: `read()` or `write()`.
- Set TTLs on cache entries (`setEx`/`ex=`) so stale data expires.
- Redis channels (`pubsub` group) are fire-and-forget to current subscribers. For durable, retried messaging use Pub/Sub topics.

## v1 keyspace to v2

```typescript
// v1
const sessions = new StringKeyspace<{ id: string }>(cluster, { keyPattern: "session/:id", defaultExpiry: expireIn(3600 * 1000) });
await sessions.set({ id }, token);

// v2
const sessions = new Database("sessions", { evictionPolicy: "allkeys-lru" });
await sessions.string.setEx(`session/${id}`, token, 3600);
```
