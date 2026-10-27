# Object storage in Go

Package: `sdk.encore.dev/storage/objects`. Same API as v1's `encore.dev/storage/objects`.

## Declare

```go
import "sdk.encore.dev/storage/objects"

var avatars = objects.NewBucket("avatars", objects.BucketConfig{Versioned: false}) // Public, Versioned default false
var images = objects.NewBucket("public-images", objects.BucketConfig{Public: true})
```

Another service's bucket: `var avatars = objects.Named("avatars")`.

## Operations

**`Upload` returns a `*Writer`: the object exists only once `Close()` returns nil.**

```go
w := avatars.Upload(ctx, "users/"+userID+".png",
	objects.WithUploadAttrs(objects.UploadAttrs{ContentType: "image/png"}))
if _, err := io.Copy(w, r.Body); err != nil {
	w.Abort(err) // or cancel ctx
	return err
}
if err := w.Close(); err != nil {
	return err
}

// Only if the key is free; Close fails with ErrPreconditionFailed otherwise.
w = avatars.Upload(ctx, key, objects.WithPreconditions(objects.Preconditions{NotExists: true}))
```

`Download` returns a `*Reader`; read it and close it. Errors surface from `Read` or `Err()`:

```go
rd := avatars.Download(ctx, key) // objects.WithRange(0, 1024), objects.WithRangeFrom(512)
defer rd.Close()
data, err := io.ReadAll(rd)
if errors.Is(err, objects.ErrObjectNotFound) {
	// 404
}
```

```go
ok, err := avatars.Exists(ctx, key)
attrs, err := avatars.Attrs(ctx, key) // Name, Size, ETag, ContentType, Version
for e, err := range avatars.List(ctx, &objects.Query{Prefix: "users/", Limit: 100}) {
	if err != nil {
		return err
	}
	fmt.Println(e.Name, e.Size)
}
err = avatars.Remove(ctx, key)
```

In a versioned bucket, `Download`, `Attrs`, `Exists` and `Remove` take `objects.WithVersion(v)`.

## Signed and public URLs

```go
up, err := avatars.SignedUploadURL(ctx, key, objects.WithTTL(10*time.Minute)) // up.URL; client PUTs to it
dl, err := avatars.SignedDownloadURL(ctx, key, objects.WithTTL(time.Hour))    // default 1h, max 7 days

u := images.PublicURL("logo.png") // *url.URL; panics unless the bucket is public
```

## Errors

`objects.ErrObjectNotFound`, `objects.ErrPreconditionFailed`, `objects.ErrInvalidArgument` (e.g. a bad range). Test with `errors.Is`.

## Passing a bucket to other code

A `*Bucket` can't be passed or assigned (parse error). Make a reference with the permissions the code needs:

```go
func resize(ctx context.Context, b objects.ReadWriter, key string) error {
	rd := b.Download(ctx, key)
	defer rd.Close()
	// ...
	w := b.Upload(ctx, "thumbs/"+key)
	// ...
	return w.Close()
}

err := resize(ctx, objects.BucketRef[objects.ReadWriter](avatars), key)
```

Permissions: `Uploader`, `SignedUploader`, `Downloader`, `SignedDownloader`, `Attrser` (`Attrs`, `Exists`), `Lister`, `Remover`, `PublicURLer`. `ReadWriter` is all of them except `PublicURLer`. Combine with an interface: `objects.BucketRef[interface{ objects.Uploader; objects.Downloader }](avatars)`.
