---
name: encore-beta-bucket
description: Store files in object storage buckets in an Encore v2 (beta) app, in TypeScript, Python, Go or Rust. Covers declaring public and versioned buckets, upload, download, list, attributes, delete, signed upload and download URLs, public CDN URLs, errors, and the `encore objects` commands.
when_to_use: >-
  User is in an Encore v2 app and wants to upload or serve files, images, avatars, documents, exports or generated reports, let a browser upload directly, or store blobs. Trigger phrases: "bucket", "object storage", "S3", "GCS", "file upload", "image upload", "profile picture", "signed URL", "presigned URL", "public URL", "CDN", "blob". SKIP for v1 apps.
---

# Object storage in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

Declare a bucket in code. Encore provisions a local emulator for `encore run`, an isolated bucket per test run, and S3 or GCS when deployed.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`).

## Rules for every language

- Declare at module/package scope. Name: a kebab-case string literal, unique in the app.
- Options: `public` (objects served through a CDN, readable by anyone) and `versioned` (keep old versions on overwrite). Both default to off and must be literals.
- Call operations on the declared variable. To hand a bucket to other code, use a typed reference with the permissions it needs, where the language has one.
- Use another service's bucket by binding it by name; the service is granted the operations it calls.
- Missing objects raise a dedicated not-found error. Map it to a 404 in the HTTP handler.

## Choosing how files move

- **Small files through your API:** upload/download through the service.
- **Large files or direct browser uploads:** return a signed upload URL (the client `PUT`s to it) or signed download URL. TTL is in seconds, default 1 hour, max 7 days. Anyone holding the URL can use it until it expires.
- **Public assets:** a `public` bucket and its public URL, served through a CDN.

## Local inspection

```bash
encore objects list-buckets
encore objects list <bucket>
encore objects cat <bucket> <key>
encore objects upload ...      # see --help
encore objects rm ...
```

Locally, signed and public URLs point at the emulator, which enforces expiry but not signatures.
