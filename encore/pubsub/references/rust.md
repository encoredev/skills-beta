# Pub/Sub in Rust

Module: `encore::pubsub`.

## Declare a topic

```rust
use encore::pubsub::Topic;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced { pub order_id: String, pub customer_id: String, pub total: i64 }

pub static ORDER_PLACED: Topic<OrderPlaced> = Topic::new("order-placed");
static PAYMENTS: Topic<OrderPlaced> = Topic::new("payments").exactly_once(); // default at-least-once
```

Bind a topic declared elsewhere (another service or language). The message type is your claim; it can't be checked across languages:

```rust
static ORDER_PLACED: Topic<OrderPlaced> = Topic::named("order-placed");
```

## Publish

```rust
let message_id: String = ORDER_PLACED.publish(&event).await?;

use encore::pubsub::PublishOptions;
let attributes = [("source".to_string(), "checkout".to_string())].into();
ORDER_PLACED.publish_with(&event, PublishOptions { attributes }).await?;
```

Payloads are JSON (`serde`). For bytes, `Topic<Vec<u8>, encore::pubsub::Raw>`. Protobuf: enable the `protobuf` feature and use `Topic<T, encore::pubsub::Protobuf>` with a `prost` message.

## Subscribe

```rust
use std::time::Duration;
use encore::pubsub::{Message, Subscription};

#[encore::pubsub::register]
static SEND_CONFIRMATION: Subscription<OrderPlaced> =
    Subscription::new(&ORDER_PLACED, "send-confirmation", send_confirmation);

pub async fn send_confirmation(msg: Message<OrderPlaced>) -> Result<(), encore::Error> {
    send_email(&msg.data.customer_id, &msg.data.order_id).await?; // Err = retried
    Ok(())
}
```

- **Starting:** `#[encore::pubsub::register]` has `encore::initialize()` (or `run_blocking()`) start the subscription. Without it, call `SEND_CONFIRMATION.start()?` yourself; a `static` does nothing until used. A subscription nothing started logs `subscription "..." has no handler` at startup, and its messages wait.
- **Handler:** with `register`, pass the `async fn` by name. Without it, the handler must box: `|msg| Box::pin(send_confirmation(msg))`.
- **Message:** `data`, `raw_data` (`Vec<u8>`), `id` (stable across redeliveries), `attributes` (`HashMap<String, String>`), `delivery_attempt` (from 1), `publish_time` (`SystemTime`), `ordering_key` (`Option<String>`). It is **not** the payload itself.
- **Decoding:** a payload that fails to deserialize is a failed delivery, retried like a handler error.

### Options (literals only)

```rust
#[encore::pubsub::register]
static SEND_CONFIRMATION: Subscription<OrderPlaced> =
    Subscription::new(&ORDER_PLACED, "send-confirmation", send_confirmation)
        .ack_deadline(Duration::from_secs(60))            // default 30s
        .message_retention(Duration::from_secs(7 * 86400)) // default 7 days
        .min_backoff(Duration::from_secs(10))              // default 10s
        .max_backoff(Duration::from_secs(600))             // default 10m
        .max_retries(20)                                   // default 100; -1 forever, -2 never retry
        .max_concurrency(20);                              // default unlimited
```

Durations must be `Duration::from_secs(...)` or `Duration::from_millis(...)`, and every number an integer literal or a product of them (`7 * 86400`): the parser reads them from the source. Anything else (a `const`, a variable) is a parse error, not a silent default.

## Ordered topics

```rust
use encore::pubsub::OrderedTopic;

static ACCOUNT_EVENTS: OrderedTopic<AccountEvent> = OrderedTopic::new("account-events");
ACCOUNT_EVENTS.publish(&event, &account_id).await?; // key required; "" fails with InvalidArgument
```

## Passing a topic to other code

Not supported in Rust: a topic can't be passed as a value, and there's no reference API yet (the parser error mentions `.reference()`, which `Topic` doesn't have). Wrap `publish` in a function next to the declaration and call that.

There is no current-request API; everything about the delivery is on `Message`.
