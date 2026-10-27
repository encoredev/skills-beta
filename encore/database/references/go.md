# Postgres in Go

Package: `sdk.encore.dev/storage/postgres`.

## Declare

```go
// users/db.go
import "sdk.encore.dev/storage/postgres"

var db = postgres.NewDatabase("users", postgres.DatabaseConfig{
	Migrations: "./migrations", // relative to the package directory, no "../"
})
```

Migrations are `<number>_<description>.up.sql`. Go has no ORM migration source option (`DatabaseConfig` has only `Migrations`).

Another service's database: `var users = postgres.Named("users")`.

## Query

Parameters are `$1`, `$2`, ... passed as args, never formatted into the SQL. Always pass the request's context.

```go
var name string
err := db.QueryRow(ctx, "SELECT name FROM users WHERE email = $1", email).Scan(&name)
if errors.Is(err, postgres.ErrNoRows) {
	// not found
}

rows, err := db.Query(ctx, "SELECT id, name FROM users ORDER BY name")
if err != nil {
	return err
}
defer rows.Close()
for rows.Next() {
	var id int64
	var name string
	if err := rows.Scan(&id, &name); err != nil {
		return err
	}
}
if err := rows.Err(); err != nil {
	return err
}

res, err := db.Exec(ctx, "DELETE FROM users WHERE id = $1", id)
n := res.RowsAffected() // int64
```

- `Rows` hold a connection until read to the end or closed. Always `defer rows.Close()`.
- `Scan` takes exactly one destination per column; `nil` skips a column.
- `Exec` discards `RETURNING` rows. Use `QueryRow` to read them.

## Errors

```go
import "sdk.encore.dev/storage/postgres/sqlerr"

if postgres.ErrCode(err) == sqlerr.UniqueViolation { /* 409 */ }

var pgErr *postgres.Error // Code, Severity, DatabaseCode (SQLSTATE), Message, TableName, ConstraintName, ...
if errors.As(err, &pgErr) { /* ... */ }
```

Codes: `UniqueViolation`, `ForeignKeyViolation`, `NotNullViolation`, `CheckViolation`, `ExcludeViolation`, `TransactionFailed`, `DeadlockDetected`, `TooManyConnections`, `Other`.

## Transactions

```go
tx, err := db.Begin(ctx)
if err != nil {
	return err
}
defer tx.Rollback() // no-op after Commit
if _, err := tx.Exec(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from); err != nil {
	return err
}
if _, err := tx.Exec(ctx, "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to); err != nil {
	return err
}
return tx.Commit()
```

A transaction never ended rolls back once garbage-collected, but don't rely on it.

## Passing the database to other code

A `*postgres.Database` can't be passed, assigned or stored: the parser rejects it. Pass a handle, typed as `postgres.Queryable` (`Exec`, `Query`, `QueryRow`):

```go
func count(ctx context.Context, q postgres.Queryable) (n int64, err error) {
	err = q.QueryRow(ctx, "SELECT count(*) FROM users").Scan(&n)
	return n, err
}

n, err := count(ctx, db.Read())
```

| Handle | Access |
|---|---|
| `db.Read()` | Read-only; may use a read replica. Writes fail with SQLSTATE `25006`. |
| `db.ReadPrimary()` | Read-only, always the primary |
| `db.Write()` | Read and write |
| `db.Admin()` | Read, write and administration |

Calling query methods on the declaration itself counts as write access. Handles (`postgres.DatabaseRef`) and `*postgres.Tx` both implement `Queryable`, and handles also have `Begin`.

## ORMs and other drivers

There's no `Stdlib()` in v2. Open your own driver with the connection string:

```go
connStr, err := db.ConnectionString()          // read-write
roStr, err := db.Read().ConnectionString()     // read-only
pool, err := pgxpool.New(ctx, connStr)         // or sql.Open("pgx", connStr), GORM, sqlc, ...
```

Queries through another driver are not traced; add that driver's OpenTelemetry instrumentation (see the observability skill).
