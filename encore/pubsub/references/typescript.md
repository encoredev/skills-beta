# Pub/Sub in TypeScript

Package: `@encore.dev/sdk/pubsub`.

## Declare a topic

```typescript
// orders/topics.ts
import { Topic } from "@encore.dev/sdk/pubsub";

export interface OrderPlaced { orderId: string; customerId: string; total: number }

export const orderPlaced = new Topic<OrderPlaced>("order-placed", {
  deliveryGuarantee: "at-least-once", // default; or "exactly-once"
});
```

Bind a topic declared elsewhere (another service or language). Give the type, it can't be inferred:

```typescript
const orderPlaced = Topic.named<OrderPlaced>("order-placed");
```

An ordered topic binds with `OrderedTopic.named<T>("name")`.

## Publish

```typescript
const messageId = await orderPlaced.publish({ orderId, customerId, total });
await orderPlaced.publish(event, { source: "checkout" }); // string attributes
```

Payloads are JSON. For bytes, declare `Topic<Buffer>` (or `Uint8Array`) and publish a `Buffer`. Always `await` publish.

## Subscribe

```typescript
// email/subscriptions.ts: must be imported at startup
import { Subscription } from "@encore.dev/sdk/pubsub";
import { orderPlaced } from "../orders/topics";

export const sendConfirmation = new Subscription(orderPlaced, "send-confirmation", {
  handler: async (message) => {
    await sendEmail(message.data.customerId, `Order ${message.data.orderId} confirmed`);
  },
});
```

Subscriptions start as soon as they're declared. There's no start call in TypeScript, but the declaring module must be imported by the service's entry point.

The handler gets a `Message<T>`: `data`, `rawData` (Buffer), `id` (stable across redeliveries), `attributes`, `deliveryAttempt` (from 1), `publishTime` (Date), `orderingKey` (or `null`). It is **not** the payload itself; v1 handlers received the payload.

### Validate at runtime

```typescript
import { z } from "zod";
const Schema = z.object({ orderId: z.string(), customerId: z.string(), total: z.number() });

new Subscription(orderPlaced, "send-confirmation", {
  decode: (data) => Schema.parse(data), // failure = failed handler, retried
  handler: async (message) => { /* message.data is validated */ },
});
```

Write callbacks as properties (`decode: (data) => ...`). The parser rejects method shorthand (`decode(data) { ... }`).

Bytes: `encoding: "raw"` makes `data` the raw `Buffer`; a `Topic<Buffer>` subscription must set it. With `encoding: "raw"`, `decode` parses a custom format from the bytes.

### Options (literals only)

```typescript
new Subscription(orderPlaced, "send-confirmation", {
  handler,
  retryPolicy: { minBackoff: "10s", maxBackoff: "10m", maxRetries: 20 }, // defaults 10s / 10m / 100
  ackDeadline: "60s",       // default 30s, min 1s
  messageRetention: "7d",   // default 7 days
  maxConcurrency: 20,       // per instance; negative = unlimited; no effect on Encore Cloud
});
```

## Ordered topics

```typescript
import { OrderedTopic } from "@encore.dev/sdk/pubsub";

export const accountEvents = new OrderedTopic<AccountEvent>("account-events");
await accountEvents.publish({ type: "deposit", amount: 100 }, accountId); // key: non-empty string, required
```

## Passing a topic to other code

```typescript
import type { Publisher } from "@encore.dev/sdk/pubsub";

async function notify(topic: Publisher<OrderPlaced>, event: OrderPlaced) {
  await topic.publish(event);
}
await notify(orderPlaced.ref<Publisher<OrderPlaced>>(), event);
```

For an ordered topic the permission is `OrderedPublisher<T>`.

Inside a handler, `currentRequest()` from `@encore.dev/sdk` returns `{ type: "pubsub-message", service, topic, subscription, messageId, deliveryAttempt, trace }`.
