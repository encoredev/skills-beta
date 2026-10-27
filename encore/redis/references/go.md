# Redis in Go

Package: `sdk.encore.dev/storage/redis`. Replaces v1's `cache.NewCluster` and keyspaces.

## Declare

```go
import "sdk.encore.dev/storage/redis"

var sessions = redis.NewDatabase("sessions", redis.DatabaseConfig{})                              // a cache: may evict
var ledger = redis.NewDatabase("ledger", redis.DatabaseConfig{Persistent: true})                 // never evicted
var pageCache = redis.NewDatabase("page-cache", redis.DatabaseConfig{EvictionPolicy: redis.AllKeysLfu}) // default AllKeysLru
```

Name: kebab-case literal. Another service's database: `redis.Named("sessions")`. Keys are plain strings; there are no keyspaces.

## Commands

Commands are grouped by data type and named after the Redis command:

```go
err := sessions.String.Set(ctx, "session:"+id, payload)
err = sessions.String.SetEx(ctx, "otp:"+userID, code, 300) // seconds
_, err = sessions.String.SetOptions(ctx, key, value, redis.SetOptions{
	Condition: redis.NX, Expiry: redis.SetExpiryEx(60),
}).Text()
visits, err := sessions.String.Incr(ctx, "visits:"+userID, 1)
_, err = sessions.Hash.Hset(ctx, "user:"+userID, "name", name)
_, err = sessions.List.Lpush(ctx, "jobs", jobID)
_, err = sessions.Set.Sadd(ctx, "tags:"+postID, "go")
_, err = sessions.SortedSet.Zadd(ctx, "leaderboard", userID, score)
_, err = sessions.Generic.Expire(ctx, "session:"+id, 3600)
_, err = sessions.Generic.Del(ctx, "session:"+id)
```

Groups: `String`, `Hash`, `List`, `Set`, `SortedSet`, `Stream`, `Geo`, `Bitmap`, `Pubsub`, `Generic`.

**Call a command in the same expression as its group** (`sessions.String.Get(...)`). Storing a group (`s := sessions.String`) is a parse error.

## Reading replies

Fixed replies come back directly (`Exists` → `bool`, `Incr` → `int64`). Value replies are a `redis.Reply`; pick the type:

```go
name, err := sessions.String.Get(ctx, "name:"+userID).Text()
if errors.Is(err, redis.ErrNil) {
	// missing key: not an infrastructure error
}
visits, err := sessions.String.Get(ctx, "visits:"+userID).Int64()
user, err := sessions.Hash.Hgetall(ctx, "user:"+userID).TextMap()
top, err := sessions.SortedSet.ZrangeWithscores(ctx, "leaderboard", 0, 9).ScoredMembers()
```

Other readers: `Bytes`, `Float64`, `Bool`, `Texts`, `Int64s`, `OptionalTexts` (for `Mget`), `Replies`, `Value`. `reply.IsNil()` tests for nil without converting. Scans are iterators: `for key, err := range sessions.Generic.Scan(ctx, redis.ScanOptions{Pattern: "user:*"})`.

## Pipelines, transactions, watch, scripts

Queued commands take no `ctx`; `Exec` does.

```go
pipe := sessions.Pipeline() // one round trip, not atomic
pipe.String.Get("name:1")
pipe.String.Get("name:2")
replies, err := pipe.Exec(ctx) // replies[i].Text(), each with its own error

tx := ledger.Transaction() // MULTI/EXEC; no rollback in Redis
tx.String.Incr("hits", 1)
_, err = tx.Exec(ctx)

err = ledger.Watch(ctx, []string{"balance"}, func(ctx context.Context, conn *redis.Conn, tx *redis.WatchedTransaction) error {
	balance, err := conn.String.Get(ctx, "balance").Int64() // optimistic, rerun on conflict
	if err != nil {
		return err
	}
	tx.String.Set("balance", balance-10)
	return nil
})

var release = redis.NewScript(`if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) end return 0`)
ok, err := locks.Script(release).Key("lock:" + job).Arg(token).Invoke(ctx).Bool()
```

Pass every key a script touches with `.Key()`: only those get the database's key prefix. `redis.NewReadOnlyScript` runs on a read handle.

## Passing the database to other code

```go
func nameOf(ctx context.Context, db redis.ReadOnlyDatabaseRef, user string) (string, error) {
	return db.String.Get(ctx, "name:"+user).Text()
}
name, err := nameOf(ctx, sessions.Read(), userID)
```

Handles: `Read()` and `ReadPrimary()` (`ReadOnlyDatabaseRef`, read commands only), `Write()` (`DatabaseRef`).

## Other clients

`sessions.ConnectionString()` works with go-redis and others, but bypasses the key prefix, so keys don't match the SDK's.
