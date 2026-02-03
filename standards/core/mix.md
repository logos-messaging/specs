---
title: WAKU-MIX
name: Waku Mix
editor: Prem Chaitanya Prathi <prem@status.im>
contributors:
  - Akshaya Mani <akshaya@status.im>
  - Hanno Cornelius <hanno@status.im>
---

## Tags

`waku/core-protocol`

# Abstract

The document describes [libp2p mix](https://lip.logos.co/ift-ts/raw/mix.html) integration into waku.
This integration provides higher anonymity for users publishing or querying for messages to/from the Waku network.

This document covers integration of mix with [lightpush](https://rfc.vac.dev/waku/standards/core/19/lightpush) and [store](https://rfc.vac.dev/waku/standards/core/13/store) protocols.
Both of these are `request-response` based protocols that follow a service-user and a service-provider model.
A node that initiates the request is a `service-user/client` whereas a node that replies to the request is the `service-provider/service-node`.

This document also covers the aspect of relay nodes acting as mix nodes.

## Background / Rationale / Motivation

Waku protocols have weak sender/originator anonymity as explained in [Waku Privacy and Anonymity Analysis](https://vac.dev/rlog/wakuv2-relay-anon/).
Without further anonymization, it is easy to analyze the network traffic and determine the originator of messages published into the network.
Topic interests of a user can be identified by analyzing [Store](https://rfc.vac.dev/waku/standards/core/13/store) query messages sent by a user.
Same applies for topic subscriptions via [Filter](https://rfc.vac.dev/waku/standards/core/12/filter) protocol.

`Mix` protocol allows libp2p nodes to send messages without revealing the sender's identity (peer ID, IP address) to intermediary mix nodes or the recipient/destination.
Anonymity is achieved by using the [Sphinx packet format](#references), which encrypts and routes messages through a series of mix nodes before reaching the recipient.

By integrating the `mix` protocol into the waku network, we can improve the anonymity for publishers and store query users.
Each waku relay node SHOULD be acting as a mix node that forms an `overlay mix network`.
This network of mix nodes SHALL relay mix messages anonymously to the recepient.

Anonymity of [Filter](https://rfc.vac.dev/waku/standards/core/12/filter) users is not addressed by this document.

## Terminology

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”,
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## Theory / Semantics

Waku Mix creates an `overlay network` of all the Waku nodes that support the `mix` protocol.

Nodes with `mix` protocol mounted SHOULD advertise that they support `mix` protocol via their their chosen discovery method.
They MAY do so by updating their [ENR](#enr-updates) and using one of the ENR based discovery methods.

Nodes that want higher anonymity while publishing a message via [lightpush](https://rfc.vac.dev/waku/standards/core/19/lightpush) or performing a [store](https://rfc.vac.dev/waku/standards/core/13/store) query SHOULD use the `mix` protocol to route their messages to the destination.
Sender Node that would like to use `mix` protocol SHOULD discover enough mix nodes so that there is always a healthy pool of mix nodes available for selection.
The pool size of mix nodes SHOULD be large enough for the mixing to be effective.
We RECOMMEND a pool size of at least 100 mix nodes for the mixing to be effective.

The serialized [Waku Message](https://rfc.vac.dev/waku/standards/core/14/message) MUST be the payload in the [sphinx packet](https://rfc.vac.dev/vac/raw/mix#4-sphinx-packet-format).

To allow acknowledgments from the `service-node` while preserving sender anonymity, `Single Use Reply Blocks` or `anonymous replies` as specified in the original [sphinx](#references) paper SHALL be used.

A node that sends messages using `mix` MAY use two redundant (paths)[https://rfc.vac.dev/vac/raw/mix#24-node-discovery] to have better reliability of the message being delivered.
It is up to the higher-layer mixed protocol to deduplicate redundant messages received in this way.

## Node Roles

Mix protocol defines 3 roles for the nodes in the mix network - `sender`, `exit`, `intermediary`.

- An `sender` node is the originator node of a message, i.e a node that wishes to publish/query messages to/from the waku network.
- An `exit` node is responsible for delivering messages to destination peer in the network.
- An `intermediary` node is responsible for forwarding a mix packet to the next mix node in the path.

A Waku relay node SHOULD by default have mix `intermediary` and `exit` node roles in the network.
The implementation MAY provide a configuration to disable a node from acting as an `intermediary\exit` node.

Any waku node that wishes to publish/query messages via `mix` from the waku network MUST act as a `sender` node.

Resource-restricted/Edge nodes with short connection windows MUST _only_ act as `sender` nodes and cannot function as intermediaries or exit nodes.

## ENR updates

Each waku node that supports the `mix intermediary or exit role` SHOULD indicate the same in its discoverable [ENR](https://github.com/waku-org/specs/blob/master/standards/core/enr.md).
The following fields MUST be set as part of the discoverable ENR of a mix waku node:

- The `bit 5` in the [waku2 ENR key](https://github.com/waku-org/specs/blob/master/standards/core/enr.md#waku2-enr-key) is reserved to indicate `mix` support. This bit MUST be set to true to indicate `mix` support.
- A new field `mix-key` SHOULD be set to the `ed25519 public key` which is used for sphinx encryption.

Adding these fields to the ENR may cause the ENR size to cross the 300 byte limit especially in case a node supports multiple transports.
This limitation will have to be addressed in future.

### Discovery

Mix protocol provides better anonymity when a sender node has a sufficiently large pool of mix nodes to do path selection.
This moves the problem into discovery domain and requires the following from discovery mechanisms:

1. It is important for nodes to be able to discover as many nodes as possible quickly. This becomes especially important for edge nodes that come online just to publish/query messages for a short period of time.
2. The discovery mechanism MUST be unbiased and not biased toward specific subsets (e.g., nodes that are topologically closer).
3. It is important to have the most recent online status of the nodes so that mix paths that are selected are not broken which lead to reliability issues.

Point-3 above can be mitigated partially by choosing redundant mix paths for the same message by the sender node.
This may not be an effective solution as it increases the overall bandwidth usage.

## Spam Protection

Mix protocol in waku network SHOULD have `rate-limiting/spam` protection to handle scenarios such as below:

1. Any node can generate a mix packet and publish into the mix network. Hence there needs to be some validation as to who is allowed to publish and whether the user is within allowed rate-limits.
2. Any node can intentionally generate paths which are broken and send messages into the mix network.
3. An attacker can spawn a huge number of mix nodes so that user behaviour is observed in order to determine traffic patterns and deanonymize users.

There is a need to enforce rate-limits and spam protect the mix network.
The rate-limiting and spam protection shall be addressed as part of future work.

## Tradeoffs

Using `mix` protocol for publishing and querying messages adds certain overhead which is primarily the delay in delivering message to the destination.
The overall additional delay `D` depends on the following params:

- path length `L`
- delay added by each intermediary node `dm`
- connection establishment time `dc`
- processing delay `dp`

Delay overhead can be calculated as `D =  L * (dm + dc +dp)`

## Future Work

- Integration of [waku relay](https://rfc.vac.dev/waku/standards/core/11/relay) with mix.
- [Spam protection](#spam-protection) of mix overlay network.
- Alternative way to communicate mix-public-key in order to overcome [ENR](#enr-updates) size limitations.

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [libp2p mix](https://lip.logos.co/ift-ts/raw/mix.html)
- [waku lightpush](https://rfc.vac.dev/waku/standards/core/19/lightpush)
- [waku relay](https://rfc.vac.dev/waku/standards/core/11/relay)
- [ENR](https://github.com/waku-org/specs/blob/master/standards/core/enr.md)
- [sphinx encryption](https://cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf)
