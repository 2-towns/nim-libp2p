# Experimental GossipSub topic streams

Topic streams implement the working draft in
[libp2p/specs#729](https://github.com/libp2p/specs/pull/729) using `/gsts/v0beta`.
They are disabled by default. Enable them explicitly when constructing GossipSub:

```nim
let parameters = GossipSubParams.init(
  topicStreamsConfig = Opt.some(TopicStreamsConfig())
)
# Pass parameters to GossipSub.init(..., parameters = parameters).
```

Supporting GossipSub 1.3 peers advertise protobuf extension field `6492435`.
Peers without this capability continue using ordinary GossipSub RPCs. After
negotiation, full and partial application messages use one outgoing stream per
topic; subscriptions and all other control traffic retain the ordinary stream.
This routing takes precedence over `PublishParams.useCustomStream` for peers
that negotiated topic streams. The draft forbids empty application payloads, so
they are not delivered to peers using topic streams.
Each direction opens its own stream. The stream header supplies the topic, which
is omitted from subsequent payloads and restored before message ID generation,
signature verification, validation, and application delivery. Partial-message
metadata requests remain valid even when they contain no application data.

Each topic has an independent writer. Locally published messages take precedence
over relays and responses within that topic. Broadcasts without observers encode
each topic payload once for all supporting peers. No topic stream state is
allocated when the option is disabled.

## Resource limits and lifecycle

`TopicStreamsConfig` applies these limits per peer:

| Setting | Default | Accounts for |
| --- | --- | --- |
| `maxStreams` | 128 | Incoming and outgoing streams, including pending headers and dials |
| `maxQueuedBytes` | 8 MiB | Queued and in-flight outgoing payloads, plus retained outgoing headers |
| `maxIncomingBytes` | 8 MiB | Incoming frames being read or handled, plus retained incoming headers |
| `maxQueuedMessages` | 1024 | Queued and in-flight outgoing application messages |

`maxStreams` must be at least 3, and the byte and message limits must be greater
than zero. Invalid limits cause GossipSub initialization to fail.

These are encoded-byte budgets, not measurements of total heap usage. Protobuf
objects, queue entries, and transport framing/buffering add bounded overhead.
The existing `maxMessageSize` also limits each header and application frame.
Outgoing overflow is dropped without changing the peer's score. Excess incoming
streams or frames are reset and add a behavior penalty.
Header reads time out after one second; outgoing stream establishment times out
after five seconds.

The receiver accepts up to three overlapping streams for a topic to tolerate
transport reordering during replacement. The sender closes an old stream before
opening its replacement. Remote unsubscribe, local unsubscribe, fanout expiry,
and peer shutdown clean up the corresponding streams. A remotely closed stream
is recreated lazily on the next send after cleanup completes. Failed or closing
streams can drop messages, as with other GossipSub transport failures.

Protocol violations disconnect the peer. The current transport-independent
connection shutdown API does not expose the draft's `0xd52505` application error
code. Incoming streams are processed as they become readable: the generic stream
API does not expose the initiator's stream-open ordering. Unexpected topic names
are not penalized, avoiding penalties for subscription races.

The implementation follows the proposal through revision
`0a5da5a28d63f36c28d4cef6eb2067f2a8147e70`. The wire protocol can change while the
proposal remains a draft. Local protocol and network tests do not substitute for
cross-implementation interoperability testing or a deployment soak test.
