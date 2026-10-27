# Pub/Sub in Python

Import: `from encore import pubsub`.

## Declare a topic

```python
# orders/events.py
from encore import pubsub
from pydantic import BaseModel

class OrderPlaced(BaseModel):
    order_id: int
    amount: int

orders = pubsub.Topic[OrderPlaced]("order-placed")             # type arg optional, for type checking only
payments = pubsub.Topic("payment-received", exactly_once=True)
```

Name: lowercase letters, digits, hyphens and periods. Bind a topic declared elsewhere: `pubsub.Topic.named("user.signed-up")` (or `OrderedTopic.named`).

## Publish

```python
message_id = await orders.publish(OrderPlaced(order_id=1, amount=100))
await signups.publish({"user_id": 42}, attributes={"source": "web"})
```

Payload: a top-level Pydantic v2 model, any JSON-serializable value, or `bytes`. A dataclass, or a model nested inside a plain dict, raises `TypeError`.

## Subscribe

```python
async def send_receipt(message: pubsub.Message[OrderPlaced]) -> None:
    print("Sending receipt for", message.data.order_id)

receipts = pubsub.Subscription[OrderPlaced](orders, "send-receipt", handler=send_receipt)
```

The subscription's **type argument** chooses the decoder (the handler annotation doesn't):

| Declaration | `message.data` |
|---|---|
| `Subscription[MyModel]` | Pydantic v2 model, validated |
| `Subscription` or `Subscription[pubsub.JSON]` | `json.loads` output, unvalidated |
| `Subscription[bytes]` | raw bytes |

Dataclasses, unions of models and string forward references aren't supported.

Message fields: `data`, `raw_data`, `attributes`, `id`, `delivery_attempt`, `publish_time` (UTC datetime), `ordering_key`.

### Start the consumers

`encore.run(app)` starts them for you (see the service skill). With your own server, **nothing is delivered until `await encore.start()` runs on the app's event loop**, after the modules declaring subscriptions are imported:

```python
import encore
import orders.events  # noqa: F401  (declares the subscriptions)

@asynccontextmanager
async def lifespan(app: FastAPI):
    await encore.start()
    yield

app = FastAPI(lifespan=lifespan)
```

Don't call it under a separate `asyncio.run()` that closes before the server starts. If a declared subscription has no running handler, Encore logs a warning shortly after startup.

### Options (literals only)

```python
pubsub.Subscription[SignupEvent](
    signups, "send-welcome-email", handler=send_welcome_email,
    ack_deadline="60s",       # default 30s
    message_retention="7d",   # default 7d
    min_backoff="5s",         # default 10s
    max_backoff="5m",         # default 10m
    max_retries=10,           # default 100; -1 forever; -2 dead-letter without retrying
    max_concurrency=20,
)
```

A non-literal value is silently ignored and the default is used.

## Ordered topics

```python
cart_events = pubsub.OrderedTopic("cart-events")
await cart_events.publish({"event": "checkout_started"}, ordering_key=str(cart_id))
```

## Passing a topic to other code

Call `publish` on the module-level variable (directly or through an import). Don't pass the topic object around.

Inside a handler, the message's processing span isn't the current OpenTelemetry span yet, so spans you create there aren't linked to the message's trace.
