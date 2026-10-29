# Cron jobs in Go

Package: `sdk.encore.dev/cron`.

```go
import "sdk.encore.dev/cron"

var _ = cron.NewJob("cleanup-expired-sessions", cron.JobConfig{
	Title: "Delete expired sessions", // optional
	Path:  "/internal/cleanup",
	Every: 1 * cron.Hour,             // or Schedule: "0 4 * * *" (UTC). One of the two.
})
```

`Every` is a constant `cron.Duration` (`10 * cron.Minute`, `6 * cron.Hour`): a multiple of a minute, at most 24 hours, and it must divide 24 hours evenly.

Serve the path as a `POST` route on your router, with the same literal:

```go
mux.HandleFunc("POST /internal/cleanup", func(w http.ResponseWriter, r *http.Request) {
	res, err := db.Exec(r.Context(), "DELETE FROM sessions WHERE expires_at < now()")
	if err != nil {
		http.Error(w, "cleanup failed", http.StatusInternalServerError)
		return
	}
	json.NewEncoder(w).Encode(map[string]int64{"deleted": res.RowsAffected()})
})
```

With Encore's API framework (`"type": "encore-api"`), serve it with a private endpoint:

```go
//encore:api method=POST path=/internal/cleanup
func (s *Service) Cleanup(ctx context.Context) error {
	_, err := db.Exec(ctx, "DELETE FROM sessions WHERE expires_at < now()")
	return err
}
```

Don't build the route from the job (`"POST " + job.Path`): the parser rejects reading a job's fields, despite the SDK's doc example.

v1 took `Endpoint: SomeAPI`; v2 takes `Path`.
