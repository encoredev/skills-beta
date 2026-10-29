---
name: encore-beta-pubsub
description: Asynchronous messaging in an Encore v2 (beta) app with Pub/Sub topics and subscriptions, in TypeScript, Python, Go or Rust. Covers declaring topics, publishing, subscriptions and the message wrapper, runtime validation, retries and dead letters, ordered topics, cross-language topics, and the `encore pubsub` commands.
when_to_use: >-
  User is in an Encore v2 app and wants to publish an event, react to one in the background, fan out to several handlers, decouple services, retry failed work, keep per-entity ordering, or inspect or replay messages locally. Also use when a subscription never receives messages. Trigger phrases: "Pub/Sub", "topic", "subscription", "publish", "event", "background job", "queue", "fan out", "ordered", "dead letter", "subscriber not receiving", "message.data". SKIP for Redis channels (`encore-beta-redis`) and for v1 apps.
---

# Pub/Sub in Encore v2

CLI commands are written `encore`; run them as `encore-beta` unless `encore version` reports 2.x (see `encore-beta-install`).

Topics are app-wide and language-neutral: a Python service can subscribe to a topic a Go service publishes. Encore provisions a local broker for `encore run`, isolated topics per test run, and SNS+SQS or GCP Pub/Sub when deployed.

**Read `references/<lang>.md`** for the service's language (`typescript`, `python`, `go`, `rust`).

## Starting subscriptions differs by language

- **TypeScript:** subscriptions start when declared. The declaring module must be imported at startup.
- **Python:** `encore.run(app)` starts them. With your own server, `await encore.start()` must run on the app's event loop.
- **Go:** `encore.Start()` must be called in `main`.
- **Rust:** `#[encore::pubsub::register]` on the subscription, then `encore::initialize()`.

A subscription that "never receives anything" is almost always a missing start call or a module that's never imported.

## Rules for every language

- Declare topics and subscriptions at module/package scope with literal names and options. Option values computed at runtime are not seen; the default is used (Rust: a parse error). Topic names are kebab-case and may contain periods (`order.placed`); subscription names are kebab-case.
- **Handlers receive a message wrapper, not the payload.** Read `.data` (also `id`, `attributes`, delivery attempt, publish time, ordering key). This is the most common v1-to-v2 mistake.
- Delivery is at-least-once by default. **Make handlers idempotent**, e.g. record processed message IDs. `exactly-once` reduces redelivery but doesn't deduplicate double publishes, and caps throughput (300 msg/s per topic on AWS).
- Type annotations don't validate incoming JSON. Validate with a decoder (TS `decode`, Python Pydantic model as the type argument).
- Always await or check the error of `publish`.
- A throwing handler is retried with backoff (default 10s to 10m, 100 retries), then dead-lettered.
- Ordered topics deliver per ordering key in publish order. A failing message blocks its key until it succeeds or is dead-lettered, so choose the retry policy deliberately.
- The subscription belongs to the service that declares it, and that service's instances handle its messages. Each subscription gets every message.
- Use another service's topic by binding it by name rather than importing its code.
- Writing to the database and publishing aren't atomic. If that matters, use a transactional outbox.

## Local inspection

```bash
encore pubsub ls                                     # topics, subscriptions, queue sizes
encore pubsub queue order-placed send-confirmation --state dead
encore pubsub show <message-id>                      # message and its delivery attempts
encore pubsub release order-placed send-confirmation # redeliver dead messages
encore pubsub publish order-placed '{"orderId": "1"}'
```

Also `history`, `pause`, `resume`, `discard`, `purge`. The dev dashboard shows each message inside the trace that published it.

## Testing

Call the handler function directly with a message you build. For end-to-end delivery tests, poll with a timeout for the side effect instead of sleeping.
