# Object storage in Python

Import: `from encore.storage import objects`.

## Declare

```python
profile_pictures = objects.Bucket("profile-pictures")                  # versioned=False, public=False
documents = objects.Bucket("documents", versioned=True)
avatars = objects.Bucket("public-avatars", public=True)
```

Another service's bucket: `objects.Bucket.named("profile-pictures")` at module scope.

## Operations

```python
attrs = await profile_pictures.upload("ada.jpeg", data, content_type="image/jpeg")
await profile_pictures.upload(key, data, if_not_exists=True)          # PreconditionFailed if taken

data = await profile_pictures.download("ada.jpeg")                    # bytes
head = await profile_pictures.download("ada.jpeg", offset=0, length=512)

await profile_pictures.exists(key)                                    # False if missing
await profile_pictures.attrs(key)                                     # name, version, size, content_type, etag
await profile_pictures.remove(key)

async for entry in profile_pictures.list(prefix="ada", limit=100):
    print(entry.name, entry.size)
```

Versioned buckets: `download`, `download_stream`, `attrs`, `exists`, `remove` take `version=`.

## Streaming

```python
from contextlib import aclosing

await videos.upload_stream("intro.mp4", chunks, content_type="video/mp4")   # async iterable of bytes

async with aclosing(videos.download_stream(src)) as chunks:                 # aclosing if you may stop early
    await videos.upload_stream(dst, chunks, content_type="video/mp4")
```

Wrap `list()` in `aclosing` too if you might break out of the loop.

## Signed and public URLs

```python
url = await profile_pictures.signed_upload_url("ada.jpeg", ttl=7200)    # client PUTs to it
url = await documents.signed_download_url("letter.pdf", ttl=7200)       # ttl 1..604800 s, default 3600
url = avatars.public_url("ada.jpeg")                                    # sync; public buckets only (parse error otherwise)
```

Unlike TypeScript, signed URL methods return the URL string directly.

## Errors

`objects.ObjectsError` (subclass of `encore.EncoreError`), with `ObjectNotFound`, `PreconditionFailed`, `InvalidArgument`:

```python
try:
    return await profile_pictures.download(f"{user_id}.jpeg")
except objects.ObjectNotFound:
    return None
```

## Passing a bucket to other code

Python has no bucket reference type yet. Call methods on the module-level variable (or an import of it). Assigning it to another variable, returning it, or passing it to a function is a parse error.
