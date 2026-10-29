# Redis in Python

Import: `from encore.storage import redis`.

## Declare

```python
sessions = redis.Database("sessions")   # kebab-case literal; no options yet (no persistence/eviction settings)
```

Python can't bind another service's Redis database by name yet. Import the declaring module's variable.

## Commands

Families are properties of the database; methods are named after the Redis command, with options as keyword arguments:

```python
await sessions.string.set("user:1", "Ada", ex=60)
await sessions.string.set(key, value, ex=60, nx=True)                       # b"OK" if written, else None
await sessions.hash.hset("user:1:profile", ("name", "Ada"), ("lang", "en"))  # tuples for grouped values
await sessions.list.rpush("queue", "job-1", "job-2")
await sessions.sorted_set.zadd("leaderboard", (100, "ada"), (85, "grace"))   # (score, member)
await sessions.string.mset(("first", "Ada"), ("last", "Lovelace"))
await sessions.set.sadd("tags", *tags)
await sessions.generic.expire("queue", 300)
await sessions.generic.delete("queue")                                       # `delete`, not `del`
```

Families: `generic`, `string`, `bitmap`, `hash`, `list`, `set`, `sorted_set`, `geo`, `stream`, `json`, `pubsub`.

Call a command directly on its family (`sessions.string.get(...)`). Storing a family in a variable is a parse error.

## Values

Store `bytes`, `str`, `int` or `float` (not `bool`). Reads return **bytes** by default; subscript the command to decode:

```python
await sessions.string.get("user:1")                         # b"Ada" or None
visits = await sessions.string.get[int]("visits")           # int | None
names = await sessions.list.lrange[str]("names", 0, -1)     # list[str]
top = await sessions.sorted_set.zrange[str]("lb", 0, 9, withscores=True)  # list[tuple[str, float]]
```

Custom types: give the class `encode(self) -> bytes` and `@classmethod decode(cls, data: bytes)`, then `get[User](...)`.

## Handles

```python
async def count_visits(db: redis.DatabaseRef, page: str) -> int:
    return await db.string.incr(f"visits:{page}")

await count_visits(sessions.write(), page)   # sessions.read() -> ReadOnlyDatabaseRef (read commands only)
```

## Not supported yet in Python

Transactions (`MULTI`/`EXEC`), Lua scripts, pipelines, replica routing, and binding by name. JSON commands need RedisJSON, which the local emulator doesn't have.

## Other clients

`sessions.connection_string` (on the database and write handles) gives full access but bypasses Encore's key namespacing, so it won't see the same keys under `encore test`.
