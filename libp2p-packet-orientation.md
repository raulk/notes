# Thoughts on packet/message orientation in libp2p

@bigs has summarised considerations eloquently here:
https://docs.google.com/document/d/1xlD26_C4aHOwE1r7uGngD1c73bcjReoDG-UCMVTJSvY/edit

This doc is a loose brain dump on the things I'd like to see implemented, and
the broad design directions I'd like us to pursue with regards to these topics.

* libp2p API for message-orientation.
* Being sympathetic with the underlying transports.
* Non-interactive protocol agreement.
* Security handshake, leveraging 0-RTT data.

I'm running out of time and battery now, but I promise to add my thoughts on
these further topics soon (tm):

* Multiplexing.
* Framing for message fragmentation, when payloads are above max. PDU size.
* Framing for shimming reliability.
* Framing for 

Motivation: I've shared these thoughts with several folks along the way (libp2p
core team members incl. @bigs, stakeholders, users), but up until now, I hadn't
recorded them in a single place.

I'm writing this fast and scrappy; I'll refine as needed.

## libp2p API for message-orientation

libp2p is currently stream-oriented in its core communication model, and the API
is tightly tied to such design choice. However, this is funny because most
protocols we've built are _de facto_ message-oriented, e.g. Identify, DHT,
pubsub, etc. The only exception I can think of is Bitswap.

We tend to delimit messages on byte streams through varint length-prefixing. We
have even created utilities to overlay message-oriented semantics on top of
ordered byte streams: [go-msgio](https://github.com/libp2p/go-msgio).

@bigs has already taken a stab at modelling what a message-oriented API could
look like. I'm sure we'll iterate as a team on the concrete aspects of that API,
as the work turns into a PoC.

What I wanted to focus on here is how this API could hook into libp2p itself.

Currently, the entrypoint of the libp2p API is the Host. I'm thinking we need a
layer of abstraction above the Host.

**libp2p should become a factory of Stacks**, of which you can build Hosts. When
obtaining a Stack, the user provides a set of feature flags that act like hints
to compose the appropriate transports, encryption channels, etc. under the hood.

Such feature flags can be:

1. Quality of service attributes: UNRELIABLE, RELIABLE.
2. Ordering attributes: ORDERED, UNORDERED.
3. Congestion control: CONGESTION_CTRL, NO_CONGESTION_CTRL.
4. Communication paradigm: message-oriented, stream-oriented.
5. Plexing: multiplexed, monoplexed.

I'd like to be able to do something like:

```go
stack := libp2p.NewStack(UNRELIABLE, UNORDERED).MessageOriented()

// Now I have a message oriented stack that has only UDP and as a transport.

host := stack.NewHost() // would return a Host with a message-oriented API.
```

## Being sympathetic with the underlying transports

UDP datagrams have a maximum length of ~64kb. They are delivered as a unit or
they're not delivered at all. And when they are delivered, they can be delivered
out of order or in duplicate.

There is no value in sending three datagrams of 1kb each, when you have all the
data available. You can send a single datagram of 3kb atomically, and avoid
deliverability considerations.

This is an interesting trade-off to have, and leads me to propose the following
guiding principles:

1. The message-oriented specialisations of our wire protocols (e.g. multiselect)
   should be non-interactive where possible. Exceptions are handshakes, which
   MUST complete in 1-RTT.
2. Where there are multiple protocols possible, do not rely on interactivity to
   negotiate one. Instead, send all possibilities upfront, and let the responder
   select one.
3. Pack as much data as possible into a datagram, as long as it doesn't exceed
   64kb.
4. Connection(less) bootstrap messages should never be fragmented.
5. For when framed messages exceed 64kb, we'll need a **reassembly protocol**.

## Non-interactive protocol agreement

multiselect 2.0 should serve as a foundation for this. Instead of interactively
agreeing on the protocol, we send all options upfront, with their corresponding
request messages. The responder picks one and responds to it. That protocol
becomes pinned to the "stream". We're essentially negotiating in-band.

This mechanism should be available in ms2.0.

Unfortunately this is subject to downgrade attacks, but I can't think of any way
to prevent it. I don't think it's possible to prevent it at all.

## Security handshake, leveraging 0-RTT data

For message-orientation, we can assume:

* we know our peer's public key ahead of time.
* the key is of an inlinable type, e.g. ed25519 or secp256k1.
* we know which secure channels the peer supports, because they're conveyed in
  the multiaddr. We don't need to negotiate the secure channel (as we do today).

Supported handshakes schemes must be 1-RTT, with 0-RTT support so we can push
payloads immediately. Rationale: we don't want to waste unreliable round trips
handshaking.

Noise is very well suited for this, possibly TLS 1.3 too, but I can't recall in
this very moment.

In Noise, assuming we use the peer's public key as the Noise static key, the
opening message of the IK handshake can contain the encrypted request payload
(framed as in _non-interactive protocol agreement_). The channel won't enjoy
perfect forward secrecy until the responder replies, but it's good enough for,
say, a DHT request.

Sidenote: we should expose what the security guarantees of each message are to
the application via an API.

@yusefnapora or I can help with explaining the details.

## Multiplexing (wip)

I think we might not need negotiable/pluggable multiplexers. But I need to think
more on this; happy to hear arguments for/against.