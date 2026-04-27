---
title: RELIABLE-CHANNEL-API
name: Reliable Channel API definition
category: Standards Track
status: raw
tags: [reliability, application, api, sds, segmentation]
editor: Logos Messaging Team
---

## Table of contents

<!-- TOC -->
  * [Table of contents](#table-of-contents)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
  * [Syntax](#syntax)
  * [API design](#api-design)
    * [Architectural position](#architectural-position)
    * [IDL](#idl)
  * [Procedures](#procedures)
    * [Outgoing message processing](#outgoing-message-processing)
    * [Incoming message processing](#incoming-message-processing)
    * [Retransmission](#retransmission)
    * [Rate limiting](#rate-limiting)
    * [Encryption](#encryption)
  * [The Reliable Channel API](#the-reliable-channel-api)
    * [Channel lifecycle](#channel-lifecycle)
    * [Channel usage](#channel-usage)
    * [Node configuration](#node-configuration)
    * [Type definitions](#type-definitions)
  * [Components](#components)
    * [Segmentation](#segmentation)
    * [Scalable Data Sync (SDS)](#scalable-data-sync-sds)
    * [Rate Limit Manager](#rate-limit-manager)
    * [Encryption Hook](#encryption-hook)
  * [Security/Privacy Considerations](#securityprivacy-considerations)
  * [Copyright](#copyright)
<!-- TOC -->

## Abstract

This document specifies the **Reliable Channel API**,
an application-level interface that sits between the application layer and the [MESSAGING-API](/standards/application/messaging-api.md) plus [P2P-RELIABILITY](/standards/application/p2p-reliability.md), i.e., `application` <-> **reliable-channel-api** <-> `messaging-api/p2p-reliability`.

It bundles segmentation, end-to-end reliability via [Scalable Data Sync (SDS)](https://lip.logos.co/ift-ts/raw/sds.html), rate limit management, and a pluggable encryption hook
into a single interface for sending and receiving messages reliably.


## Motivation

The [MESSAGING-API](/standards/application/messaging-api.md) provides peer-to-peer reliability via [P2P-RELIABILITY](/standards/application/p2p-reliability.md),
but does not provide high end-to-end delivery guarantees from sender to recipient.

This API addresses that gap by introducing:
- **[SEGMENTATION-API](/standards/application/segmentation.md)** to handle large messages exceeding network size limits.
- **[SDS](https://lip.logos.co/ift-ts/raw/sds.html)** to provide causal-history-based end-to-end acknowledgement and retransmission.
- **Rate Limit Manager** to comply with [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) constraints when sending segmented messages.
- **Encryption Hook** to allow upper layers to provide a pluggable encryption mechanism. The main motivation for encryption is to protect the payload. 

The separation between Reliable Channels and encryption ensures the API remains agnostic to identity and key management concerns,
which are handled by higher layers.

## Syntax

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## API design

### Architectural position

The Reliable Channel API sits between the application layer and the Messaging API, as follows:

```
┌────────────────────────────────────────────────────────────┐
│                   Application Layer                        │
└───────────────────────────┬────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────┐
│                 Reliable Channel API                       │
│  ┌──────────────┐ ┌─────┐ ┌───────────────┐ ┌──────────┐   │
│  │ Segmentation │ │ SDS │ │ Rate Limit Mgr│ │Encryption│   │
│  │              │ │     │ │               │ │   Hook   │   │
│  └──────────────┘ └─────┘ └───────────────┘ └──────────┘   │
└───────────────────────────┬────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────┐
│                    Messaging API                           │
│      (P2P Reliability, Relay, Filter, Lightpush, Store)    │
└────────────────────────────────────────────────────────────┘
```

### IDL

A custom Interface Definition Language (IDL) in YAML is used, consistent with [MESSAGING-API](/standards/application/messaging-api.md).

## Procedures

### Outgoing message processing

When `send` is called, the implementation MUST process `message` in the following order:

1. **Segment**: Split the payload into chunks as defined in [SEGMENTATION-API](./segmentation-api.md).
2. **Apply [SDS](https://lip.logos.co/ift-ts/raw/sds.html)**: Register each chunk with the SDS layer to track acknowledgements and enable retransmission.
3. **Encrypt**: If an `Encryption` implementation is provided, encrypt each chunk before transmission.
4. **Rate Limit**: If `RateLimitConfig.enabled` is `true`, delay dispatch as needed to comply with [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) epoch constraints.
5. **Dispatch**: Send each chunk via the underlying [MESSAGING-API](/standards/application/messaging-api.md).

### Incoming message processing

When a chunk is received from the network, the implementation MUST process it in the following order:

1. **Decrypt**: If an `Encryption` implementation is provided, decrypt the chunk.
2. **Apply [SDS](https://lip.logos.co/ift-ts/raw/sds.html)**: Deliver the chunk to the SDS layer, which emits acknowledgements and detects gaps.
3. **Reassemble**: Once all chunks for a message have been received, reassemble and emit a `reliable:message:received` event.

### Retransmission

A certain Reliable Channel MUST retransmit unacknowledged chunks after `acknowledgementTimeoutMs`,
according to the [SDS](https://lip.logos.co/ift-ts/raw/sds.html) channel state,
up to `maxRetransmissions` times.
If a chunk is not acknowledged after all retransmission attempts, a `reliable:message:send-error` event MUST be emitted.

### Rate limiting

When `RateLimitConfig.enabled` is `true`, the implementation MUST space chunk transmissions
to comply with the [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) epoch constraints defined in `epochSizeMs`.
Chunks MUST NOT be sent at a rate that would violate the RLN message rate limit for the active epoch.

### Encryption

The `encryption` parameter in `createReliableChannel` is intentionally optional.
The Reliable Channel API is agnostic to encryption mechanisms.

When an `Encryption` implementation is provided, it MUST be applied as described in [Outgoing message processing](#outgoing-message-processing) and [Incoming message processing](#incoming-message-processing).

## The Reliable Channel API

This API considers the types defined by [MESSAGING-API](/standards/application/messaging-api.md) plus the following.

### Channel lifecycle

This point assumes that a WakuNode instance is created beforehand. See `createNode` function
in [MESSAGING-API](/standards/application/messaging-api.md).

```yaml
functions:

  createReliableChannel:
    description: "Creates a reliable channel over the given content topic. Sets up the required SDS state,
    segmentation, and encryption, and subscribes to `contentTopic`."
    parameters:
      - name: node
        type: WakuNode
        description: "The underlying messaging node, as defined in [MESSAGING-API](/standards/application/messaging-api.md).
        Used to send chunks and to subscribe/unsubscribe to the content topics."
      - name: channelId
        type: string
        description: "Unique identifier for this channel. Represents the reliable (SDS), segmented, and optionally-encrypted session."
      - name: contentTopic
        type: string
        description: "The topic this channel listens and sends on. This has routing and filtering connotations."
      - name: senderId
        type: string
        description: "An identifier for this sender. SHOULD be unique and persisted between sessions."
      - name: encryption
        type: optional<Encryption>
        default: none
        description: "Optional pluggable encryption implementation. If none, messages are sent unencrypted."
    returns:
      type: result<ReliableChannel, error>

  closeChannel:
    description: "Closes a reliable channel, releases all associated resources and internal state,
    and unsubscribes from its content topic via the underlying [MESSAGING-API](/standards/application/messaging-api.md)."
    parameters:
      - name: channel
        type: ReliableChannel
        description: "The channel handle returned by `createReliableChannel`."
    returns:
      type: result<void, error>
```

### Channel usage

```yaml
functions:
  send:
    description: "Send a message through a reliable channel. The message is always segmented,
    SDS-tracked, rate-limited, and encrypted (if configured)."
    parameters:
      - name: channel
        type: ReliableChannel
        description: "The channel handle returned by `createReliableChannel`."
      - name: message
        type: array<byte>
        description: "The raw message payload to send."
    returns:
      type: result<ReliableSendId, error>
      description: "Returns a `ReliableSendId` that callers can use to correlate subsequent `MessageSentEvent` or `MessageSendErrorEvent` events."

  onMessageReceived:
    description: "Subscribes a callback to be invoked when a complete message has been received and reassembled."
    parameters:
      - name: channel
        type: ReliableChannel
        description: "The channel handle returned by `createReliableChannel`."
      - name: callback
        type: function
        description: "Invoked on each received message."
        parameters:
          - name: event
            type: MessageReceivedEvent
    returns:
      type: result<void, error>

  onMessageSent:
    description: "Subscribes a callback to be invoked when all chunks of a message have been transmitted to the network.
    This confirms network-level dispatch but does not guarantee the recipient has processed the message.
    For end-to-end confirmation, see `onMessageDelivered`."
    parameters:
      - name: channel
        type: ReliableChannel
        description: "The channel handle returned by `createReliableChannel`."
      - name: callback
        type: function
        description: "Invoked when all chunks of a send operation are acknowledged by the network."
        parameters:
          - name: event
            type: MessageSentEvent
    returns:
      type: result<void, error>

  onMessageDelivered:
    description: "Subscribes a callback to be invoked when the recipient has confirmed receipt of a message via SDS acknowledgements.
    This event is emitted asynchronously, after `MessageSentEvent`, once the SDS layer receives end-to-end acknowledgements from the recipient."
    parameters:
      - name: channel
        type: ReliableChannel
        description: "The channel handle returned by `createReliableChannel`."
      - name: callback
        type: function
        description: "Invoked when the recipient confirms end-to-end delivery."
        parameters:
          - name: event
            type: MessageDeliveredEvent
    returns:
      type: result<void, error>

  onMessageSendError:
    description: "Subscribes a callback to be invoked when a send operation fails after exhausting retransmission attempts."
    parameters:
      - name: channel
        type: ReliableChannel
        description: "The channel handle returned by `createReliableChannel`."
      - name: callback
        type: function
        description: "Invoked when a send operation fails."
        parameters:
          - name: event
            type: MessageSendErrorEvent
    returns:
      type: result<void, error>
```

### Node configuration

This spec extends `NodeConfig`, needed to create a node, which is 
defined in [MESSAGING-API](/standards/application/messaging-api.md), 
with `sds_config` and `rate_limit_config` fields.

```yaml
NodeConfig:  # Extends NodeConfig defined in MESSAGING-API
  fields:
    sds_config:
      type: SdsConfig
      description: "SDS configuration. See SdsConfig defined in this spec."
    rate_limit_config:
      type: RateLimitConfig
      description: "Rate limiting configuration, including RLN-specific attributes. See RateLimitConfig defined in this spec."
```

### Type definitions

```yaml
types:

  ReliableChannel:
    type: object
    description: "A handle representing an open reliable channel.
    Returned by `createReliableChannel` and used to send messages and receive events.
    Internal state (SDS, segmentation, encryption) is managed by the implementation."


  MessageEvents:
    type: event_emitter
    description: "Event source for reliable message events on a channel"
    events:
      "reliable:message:received":
        type: MessageReceivedEvent
      "reliable:message:sent":
        type: MessageSentEvent
      "reliable:message:delivered":
        type: MessageDeliveredEvent
      "reliable:message:send-error":
        type: MessageSendErrorEvent

  MessageReceivedEvent:
    type: object
    description: "Event emitted when a complete message has been received and reassembled."
    fields:
      message:
        type: array<byte>
        description: "The reassembled message payload."

  MessageSentEvent:
    type: object
    description: "Event emitted when all chunks of a message have been transmitted to the network.
    This confirms network-level dispatch only; it does not guarantee the recipient has processed the message.
    For end-to-end confirmation, listen for `MessageDeliveredEvent`."
    fields:
      requestId:
        type: ReliableSendId
        description: "The identifier of the `send` operation whose chunks have all been dispatched to the network."

  MessageDeliveredEvent:
    type: object
    description: "Event emitted when the recipient has confirmed end-to-end receipt of a message via SDS acknowledgements.
    This event is fired asynchronously after `MessageSentEvent`, once the SDS layer receives explicit acknowledgements from the recipient."
    fields:
      requestId:
        type: ReliableSendId
        description: "The identifier of the `send` operation confirmed as delivered by the recipient."

  MessageSendErrorEvent:
    type: object
    description: "Event emitted when a message send operation fails after exhausting retransmission attempts."
    fields:
      requestId:
        type: ReliableSendId
        description: "The identifier of the `send` operation that failed after exhausting retransmission attempts."
      error:
        type: string
        description: "Error message describing what went wrong"

  ReliableSendId:
    type: string
    description: "Unique identifier for a single `send` operation on a reliable channel.
    It groups all chunks produced by segmenting one message, so callers can correlate
    acknowledgement and error events back to the original send call.
    Internally, each chunk is dispatched as an independent [MESSAGING-API](/standards/application/messaging-api.md) call,
    producing one `RequestId` (as defined in [MESSAGING-API](/standards/application/messaging-api.md)) per chunk.
    A single `ReliableSendId` therefore maps to one or more underlying `RequestId` values,
    one per chunk sent."

  SdsConfig:
    type: object
    description: Scalable Data Sync config items.
    fields:
      persistence:
        type: Persistence
        description: "Backend for persisting the SDS local history. Implementations MAY support custom backends."
      acknowledgementTimeoutMs:
        type: uint
        default: 5000
        description: "Time in milliseconds to wait for acknowledgement before retransmitting."
      maxRetransmissions:
        type: uint
        default: 5
        description: "Maximum number of retransmission attempts before considering delivery failed."
      causalHistorySize:
        type: uint
        default: 2
        description: "Number of message IDs to consider in the causal history. With longer value, a stronger correctness is guaranteed but it requires higher bandwidth and memory."

  RateLimitConfig:
    type: object
    description: Rate limiting configuration, containing RLN-specific attributes.
    fields:
      enabled:
        type: bool
        default: false
        description: "Whether rate limiting is enforced. SHOULD be true when RLN is active."
      epochSizeMs:
        type: uint
        default: 600000  # 10 minutes
        description: "The epoch size used by the RLN relay, in milliseconds."

  Encryption:
    type: object
    description: "Interface for a pluggable encryption mechanism.
    When provided as a parameter to `createReliableChannel`, the API consumer MUST implement both encrypt and decrypt operations.
    Implementations MAY use different signatures than those described below, as long as each operation accepts a byte array and returns a byte array."
    fields:
      encrypt:
        type: function
        description: "Encrypts a byte payload. Returns the encrypted payload."
        parameters:
          - name: content
            type: array<byte>
        returns:
          type: result<array<byte>, error>
      decrypt:
        type: function
        description: "Decrypts a byte payload. Returns the decrypted payload."
        parameters:
          - name: payload
            type: array<byte>
        returns:
          type: result<array<byte>, error>

  Persistence:
    type: object
    description: "Interface for a pluggable SDS persistence backend.
    Implementations MUST provide all functions required to save and retrieve SDS state per channel. Implementations MUST also provide the persistence method of interest, e.g., SQLite, custom encrypted storage, etc.
    Refer to the [SDS spec](https://lip.logos.co/ift-ts/raw/sds.html) for the full definition of what state must be persisted."
```

## Components

### Segmentation

See [SEGMENTATION-API](./segmentation-api.md).

### Scalable Data Sync (SDS)

[SDS](https://lip.logos.co/ift-ts/raw/sds.html) provides end-to-end delivery guarantees using causal history tracking.

- Each sent chunk is registered in an outgoing buffer.
- The recipient sends acknowledgements back to the sender upon receiving chunks.
- The sender removes acknowledged chunks from the outgoing buffer.
- Unacknowledged chunks are retransmitted after `acknowledgementTimeoutMs`.
- SDS state MUST be persisted using the `persistence` backend configured in `SdsConfig`.

### Rate Limit Manager

The Rate Limit Manager ensures compliance with [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) rate constraints.

- It tracks how many messages have been sent in the current epoch (only the first chunk of each message counts toward the rate limit; subsequent chunks are exempt).
- When the limit is approached, chunk dispatch MUST be delayed to the next epoch.
- The epoch size MUST match the `epochSizeMs` configured in `RateLimitConfig`.

### Encryption Hook

The Encryption Hook provides a pluggable interface for upper layers to inject encryption.

- The hook is optional; when not provided, messages are sent unencrypted.
- Encryption is applied per chunk, after segmentation and SDS registration.
- Decryption is applied per chunk, before SDS delivery.
- The `Encryption` interface MUST be implemented by the caller.
- The Reliable Channel API MUST NOT impose any specific encryption scheme.

## Security/Privacy Considerations

- This API does not provide confidentiality by default. An `Encryption` implementation MUST be supplied when confidentiality is required.
- Chunk metadata (message ID, chunk index, total chunks) is visible to network observers unless encrypted by the hook.
- SDS acknowledgement messages are sent over the same content topic and are subject to the same confidentiality concerns.
- Rate limiting compliance is required to avoid exclusion from the network by RLN-enforcing relays.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
