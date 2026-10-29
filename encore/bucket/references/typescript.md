# Object storage in TypeScript

Package: `@encore.dev/sdk/storage/objects`. Same API as v1's `encore.dev/storage/objects`.

## Declare

```typescript
import { Bucket } from "@encore.dev/sdk/storage/objects";

export const avatars = new Bucket("avatars", { versioned: false }); // options: public, versioned (both default false)
export const images = new Bucket("public-images", { public: true });
```

Another service's bucket: `const avatars = Bucket.named("avatars");`

## Operations

```typescript
import { ObjectNotFound, PreconditionFailed } from "@encore.dev/sdk/storage/objects";

const attrs = await avatars.upload(`users/${userId}.png`, buffer, { contentType: "image/png" });
await avatars.upload(key, buffer, { preconditions: { notExists: true } }); // throws PreconditionFailed if taken

try {
  const data = await avatars.download(key);                                   // Buffer
  const head = await avatars.download(key, { range: { offset: 0, length: 1024 } });
} catch (err) {
  if (err instanceof ObjectNotFound) { /* 404 */ }
  throw err;
}

await avatars.exists(key);       // boolean
await avatars.attrs(key);        // name, size, etag, contentType, version
for await (const e of avatars.list({ prefix: "users/", limit: 100 })) console.log(e.name, e.size); // list({}) for all
await avatars.remove(key);
```

In a versioned bucket, `download`, `attrs`, `exists` and `remove` take `{ version }`.

## Signed and public URLs

```typescript
const { url } = await avatars.signedUploadUrl(key, { ttl: 600 });        // client PUTs to it
const { url: dl } = await avatars.signedDownloadUrl(key, { ttl: 3600 }); // ttl seconds; default 1h, max 7 days

const publicUrl = images.publicUrl("logo.png"); // throws unless the bucket is public
```

## Errors

All subclass `ObjectsError`: `ObjectNotFound`, `PreconditionFailed`, `InvalidArgument` (e.g. a range past the end).

## Passing a bucket to other code

```typescript
import type { Uploader, Downloader } from "@encore.dev/sdk/storage/objects";

async function resize(bucket: Uploader & Downloader, key: string) {
  const original = await bucket.download(key);
  await bucket.upload(`thumbs/${key}`, await makeThumbnail(original));
}
await resize(avatars.ref<Uploader & Downloader>(), key);
```

Permissions: `Uploader`, `SignedUploader`, `Downloader`, `SignedDownloader`, `Attrser` (`attrs`, `exists`), `Lister`, `Remover`, `PublicUrler`. `ReadWriter` is all of them except `PublicUrler`.
