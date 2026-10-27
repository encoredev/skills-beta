# Object storage in Rust

Module: `encore::objects`.

## Declare

```rust
use encore::objects::Bucket;

pub static AVATARS: Bucket = Bucket::new("avatars");                   // options: .public(), .versioned()
pub static IMAGES: Bucket = Bucket::new("public-images").public();
```

Another service's bucket: `static AVATARS: Bucket = Bucket::named("avatars");`

## Operations

Each method has a `_with` variant taking options; the plain one uses the defaults.

```rust
use encore::objects::{ByteRange, DownloadOptions, ListOptions, UploadOptions, UploadPreconditions};
use encore::ErrCode;

let attrs = AVATARS.upload_with(&format!("users/{user_id}.png"), bytes, UploadOptions {
    content_type: Some("image/png".into()),
    ..Default::default()
}).await?;                                            // bytes: Vec<u8>
AVATARS.upload_with(&key, bytes, UploadOptions {
    preconditions: Some(UploadPreconditions { not_exists: Some(true) }), // FailedPrecondition if taken
    ..Default::default()
}).await?;

match AVATARS.download(&key).await {                  // Vec<u8>
    Ok(data) => { /* ... */ }
    Err(e) if e.code == ErrCode::NotFound => { /* 404 */ }
    Err(e) => return Err(e),
}
let head = AVATARS.download_with(&key, DownloadOptions {
    range: Some(ByteRange { offset: 0, length: Some(1024) }),
    ..Default::default()
}).await?;

AVATARS.exists(&key).await?;                          // bool
AVATARS.attrs(&key).await?;                           // name, size, etag, content_type, version
let mut entries = AVATARS.list(ListOptions { prefix: Some("users/".into()), ..Default::default() }).await?;
while let Some(entry) = entries.next().await { let e = entry?; println!("{} {}", e.name, e.size); }
AVATARS.remove(&key).await?;
```

- **Streaming:** `upload_stream(key, impl AsyncRead, opts)`, `upload_chunks(key, stream_of_bytes, opts)`, `download_stream(key, opts)` (a `Download`: `.next()`, `.bytes()`, `.into_async_read()`).
- **Versioned buckets:** `download_with`, `attrs_with`, `exists_with` and `remove_with` take `version: Some(..)`.

## Signed and public URLs

```rust
use std::time::Duration;
use encore::objects::SignedUrlOptions;

let up = AVATARS.signed_upload_url_with(&key, SignedUrlOptions { ttl: Some(Duration::from_secs(600)) }).await?;
let dl = AVATARS.signed_download_url(&key).await?;  // default 1h, max 7 days
// up.url, dl.url: String

let url = IMAGES.public_url("logo.png")?;           // String
```

`public_url` on a bucket not declared `.public()` is a parse error (and `FailedPrecondition` at runtime).

## Errors

`encore::Error` with `code`: `NotFound` (missing object), `FailedPrecondition` (precondition failed), `InvalidArgument` (e.g. a bad range), `Unavailable` (backend errors).

## Passing a bucket to other code

A `Bucket` can't be passed as a value. Make a `BucketRef` with the permissions the code needs; library code takes the permission traits:

```rust
use encore::objects::{perms, BucketRef, Downloader, Uploader};

async fn resize(bucket: &(impl Uploader + Downloader), key: &str) -> Result<(), encore::Error> {
    let original = bucket.download(key).await?;
    bucket.upload(&format!("thumbs/{key}"), make_thumbnail(&original)).await?;
    Ok(())
}

resize(&AVATARS.reference::<(perms::Uploader, perms::Downloader)>(), &key).await?;
static UPLOADER: BucketRef<perms::Uploader> = AVATARS.reference::<perms::Uploader>(); // const
```

Permissions (`perms::X` for the type argument, trait `X` for bounds): `Uploader`, `SignedUploader`, `Downloader`, `SignedDownloader`, `Attrser` (`attrs`, `exists`), `Lister`, `Remover`, `PublicUrler`. `ReadWriter` is all of them except `PublicUrler`. Combine with a tuple.
