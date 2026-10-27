# Pub/Sub in Go

Package: `sdk.encore.dev/pubsub`.

## Declare a topic

```go
// orders/topics.go
import "sdk.encore.dev/pubsub"

type OrderPlaced struct {
	OrderID    string  `json:"order_id"`
	CustomerID string  `json:"customer_id"`
	Total      float64 `json:"total"`
}

var OrderPlacedTopic = pubsub.NewTopic[*OrderPlaced]("order-placed", pubsub.TopicConfig{
	ExactlyOnce: false, // default: at-least-once
})
```

Bind a topic declared elsewhere (another service or language). Give the type, it can't be inferred:

```go
var orderPlaced = pubsub.Named[*OrderPlaced]("order-placed")
```

Encoding follows the type: `[]byte` as is, a protobuf message (`proto.Message` pointer) as protobuf, anything else as JSON.

## Publish

```go
id, err := OrderPlacedTopic.Publish(ctx, &OrderPlaced{OrderID: id, CustomerID: cid, Total: total})
if err != nil {
	return err
}
_, err = OrderPlacedTopic.Publish(ctx, event, pubsub.Attributes{"source": "checkout"})
```

Always check the error. Publish with the request's context: it carries the trace and Request Context to the handler.

## Subscribe

```go
// email/subscriptions.go: its package must be imported by main
var _ = pubsub.NewSubscription(orders.OrderPlacedTopic, "send-confirmation",
	pubsub.SubscriptionConfig[*orders.OrderPlaced]{
		Handler: sendConfirmation,
	})

func sendConfirmation(ctx context.Context, msg pubsub.Message[*orders.OrderPlaced]) error {
	return sendEmail(ctx, msg.Data.CustomerID, "Order "+msg.Data.OrderID+" confirmed")
}
```

**Nothing is delivered until `main` calls `encore.Start()`** (see the service skill). Returning `nil` acks; returning an error or panicking retries.

`pubsub.Message[T]` fields: `Data` (decoded), `RawData`, `ID` (stable across redeliveries), `Attributes`, `DeliveryAttempt` (from 1), `PublishTime`, `OrderingKey` (empty on plain topics). v1 handlers received the payload; v2 handlers receive the message.

A payload that fails to decode is logged and retried like a failed handler. There's no `decode` hook in Go: validate `msg.Data` in the handler.

### Options (constants only)

```go
pubsub.SubscriptionConfig[*OrderPlaced]{
	Handler:          sendConfirmation,
	AckDeadline:      60 * time.Second,  // default 30s
	MessageRetention: 7 * 24 * time.Hour, // default 7 days
	MinBackoff:       10 * time.Second,  // default 10s
	MaxBackoff:       10 * time.Minute,  // default 10m
	MaxRetries:       20,                // default 100; pubsub.InfiniteRetries, pubsub.NoRetries
	MaxConcurrency:   20,                // per instance; default chosen by the cloud
}
```

`MaxConcurrency` must be non-negative: the parser rejects `-1` even though the SDK comment says negative means unlimited. Leave it out for the default.

## Ordered topics

```go
var AccountEvents = pubsub.NewOrderedTopic[*AccountEvent]("account-events", pubsub.TopicConfig{})

_, err := AccountEvents.Publish(ctx, &AccountEvent{Type: "deposit", Amount: 100}, accountID) // key: non-empty, required
```

Subscribe with `pubsub.NewSubscription` as usual. Bind one declared elsewhere with `pubsub.NamedOrdered[T]("account-events")`.

## Passing a topic to other code

A `*Topic` can't be passed or assigned (parse error). Make a reference:

```go
var publisher = pubsub.TopicRef[pubsub.Publisher[*OrderPlaced]](OrderPlacedTopic)

func notify(ctx context.Context, p pubsub.Publisher[*OrderPlaced], e *OrderPlaced) error {
	_, err := p.Publish(ctx, e)
	return err
}
```

For an ordered topic: `pubsub.OrderedTopicRef[pubsub.OrderedPublisher[T]](topic)`.
