# Project STAN

STAN is an extremely performant, lightweight reliable streaming platform powered by [NATS](https://nats.io).

[![License MIT](https://img.shields.io/npm/l/express.svg)](http://opensource.org/licenses/MIT) 
[![Build Status](https://travis-ci.com/nats-io/stan.svg?token=UGjrGa8sFWGQcHSJeAvp&branch=master)](http://travis-ci.com/nats-io/stan)

[Project Design Document](https://docs.google.com/document/d/1keDwK35YQnOXXKKy2HVV2oOnvEUPFyypT-Tplh8F89c/edit)

STAN provides the following high-level feature set:
- Log based.
- At-Least-Once Delivery model, giving reliable message delivery.
- Rate matched on a per subscription basis.
- Replay/Restart
- Last Value Semantics

## Notes and Known Issues for STAN Preview

- For the Preview, `stan-server` is provided in binary form [here](https://github.com/nats-io/stan-server-preview/releases)
- Please report all issues via the [Issue Tracker](https://github.com/nats-io/stan/issues)
- Please direct all questions to the #stan-preview channel on natsio.slack.com, or simply raise issues in the issue tracker
- Time- and sequence-based subscriptions are exact. Requesting a time or seqno before the earliest stored message for a subject will result in an error (in SubscriptionRequest.Error)

## Installation

```bash
# Go client
go get github.com/nats-io/stan

# Server
go get github.com/nats-io/stan-server
```

## Basic Usage

```go

sc, _ := stan.Connect(clusterID, clientID)

// Simple Synchronous Publisher
sc.Publish("foo", []byte("Hello World")) // does not return until an ack has been received from STAN

// Simple Async Subscriber
sc.Subscribe("foo", func(m *nats.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
})

// Unsubscribe
sub.Unsubscribe()

// Close connection
sc, _ := stan.Connect(clusterID, clientID)
sc.Close()
```

### Subscription Start (i.e. Replay) Options 

STAN subscriptions are similar to NATS subscriptions, but clients may start their subscription at an earlier point in the message stream, allowing them to receive messages that were published before this client registered interest. 
The options are described with examples below:

```go

// Subscribe starting with most recently published value
sub, err := sc.Subscribe("foo", func(m *stan.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, StartWithLastReceived())

// Receive all stored values in order
sub, err := sc.Subscribe("foo", func(m *stan.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, DeliverAllAvailable())

// Receive messages starting at a specific sequence number
sub, err := sc.Subscribe("foo", func(m *stan.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, StartAtSequence(22))

// Subscribe starting at a specific time
var startTime time.Time
...
sub, err := sc.Subscribe("foo", func(m *stan.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, StartAtTime(startTime))

// Subscribe starting a specific amount of time in the past (e.g. 30 seconds ago)
sub, err := sc.Subscribe("foo", func(m *stan.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, StartAtTimeDelta(time.ParseDuration("30s")))
```

### Durable Subscriptions

Replay of messages offers great flexibility for clients wishing to begin processing at some earlier point in the data stream. However, some clients just need to (e.g.) pick up where they left off from an earlier session. Durable subscriptions allow clients to assign a durable name to a subscription when it is created. Doing this causes the STAN server to track the last acknowledged message for that clientID + durable name.

```go
sc, _ := stan.Connect("test-cluster", "client-123")

// Subscribe with durable name
sc.Subscribe("foo", func(m *nats.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, DurableName("my-durable"))
...
// client receives message sequence 1-40 
...
// client disconnects for an hour
...
// client reconnects with same clientID "client-123"
sc, _ := stan.Connect("test-cluster", "client-123")

// client re-subscribes to "foo" with same durable name "my-durable"
sc.Subscribe("foo", func(m *nats.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, DurableName("my-durable"))
...
// client receives messages 41-current
```

## Advanced Usage 

### Asynchronous Publishing

The basic publish API (`Publish(subject, payload)`) is synchronous; it does not return control to the caller until the STAN server has acknowledged receipt of the message. To accomplish this, a [NUID](https://github.com/nats-io/nuid) is generated for the message on creation, and the client library waits for a publish acknowledgement from the server with a matching NUID before it returns control to the caller, possibly with an error indicating that the operation was not successful due to some server problem or authorization error.

Advanced users may wish to process these publish acknowledgements manually to achieve higher publish throughput by not waiting on individual acknowledgements during the publish operation. An asynchronous publish API is provided for this purpose:

```go
    ackHandler := func(ackedNuid string, err error) {
        if err != nil {
            log.Printf("Warning: error publishing msg id %s: %v\n, ackedNuid, err.Error())
        } else {
            log.Printf("Received ack for msg id %s\n", ackedNuid)
        }
    }
    
    // can also use PublishAsyncWithReply(subj, replysubj, payload, ah)
    nuid, err := sc.PublishAsync("foo", []byte("Hello World"), ackHandler) // returns immediately
    if err != nil {
        log.Printf("Error publishing msg %s: %v\n", nuid, err.Error())
    }
```

### Message Acknowledgements and Redelivery

STAN offers At-Least-Once delivery semantics, meaning that once a message has been delivered to an eligible subscriber, if an acknowledgement is not received within the configured timeout interval, STAN will attempt redelivery of the message. 
This timeout interval is specified by the subscription option `AckWait`, which defaults to 30 seconds.

By default, messages are automatically acknowledged by the STAN client library after the subscriber's message handler is invoked. However, there may be cases in which the subscribing client wishes to accelerate or defer acknowledgement of the message. 
To do this, the client must set manual acknowledgement mode on the subscription, and invoke `Ack()` on the `Msg`. ex:

```go
// Subscribe with manual ack mode, and set AckWait to 60 seconds
sub, err := sc.Subscribe("foo", func(m *stan.Msg) {
    m.Ack() // ack message before performing I/O intensive operation
    ...
    fmt.Printf("Received a message: %s\n", string(m.Data))
}, SetManualAckMode(), AckWait(time.ParseDuration("60s")))
```

## Rate limiting/matching

A classic problem of publish-subscribe messaging is matching the rate of message producers with the rate of message consumers. 
Message producers can often outpace the speed of the subscribers that are consuming their messages. 
This mismatch is commonly called a "fast producer/slow consumer" problem, and may result in dramatic resource utilization spikes in the underlying messaging system as it tries to buffer messages until the slow consumer(s) can catch up.

### Publisher rate limiting

STAN provides a connection option called `MaxPubAcksInFlight` that effectively limits the number of unacknowledged messages that a publisher may have in-flight at any given time. When this maximum is reached, further `PublishAsync()` calls will block until the number of unacknowledged messages falls below the specified limit. ex:

```go
sc, _ := stan.Connect(clusterID, clientID, MaxPubAcksInFlight(25))

ah := func(nuid string, err error) {
    // process the ack
    ...
}

for i := 1; i < 1000; i++ {
    // If the server is unable to keep up with the publisher, the number of oustanding acks will eventually 
    // reach the max and this call will block
    guid, _ := sc.PublishAsync("foo", []byte("Hello World"), ah) 
}
```

### Subscriber rate limiting

Rate limiting may also be accomplished on the subscriber side, on a per-subscription basis, using a subscription option called `MaxInFlight`. 
This option specifies the maximum number of outstanding acknowledgements (messages that have been delivered but not acknowledged) that STAN will allow for a given subscription. 
When this limit is reached, STAN will suspend delivery of messages to this subscription until the number of unacknowledged messages falls below the specified limit. ex:

```go
// Subscribe with manual ack mode and a max in-flight limit of 25
i := 0
sc.Subscribe("foo", func(m *nats.Msg) {
    fmt.Printf("Received message #%d: %s\n", ++i, string(m.Data))
    ...
    // Does not ack, or takes a very long time to ack
    ...
    // Message delivery will suspend when the number of unacknowledged messages reaches 25
}, SetManualAckMode(), MaxInFlight(25))

```

## License

(The MIT License)

Copyright (c) 2012-2016 Apcera Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to
deal in the Software without restriction, including without limitation the
rights to use, copy, modify, merge, publish, distribute, sublicense, and/or
sell copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
IN THE SOFTWARE.




