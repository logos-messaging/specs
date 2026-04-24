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
    * [IDL](#idl)
    * [Architectural position](#architectural-position)
  * [The Reliable Channel API](#the-reliable-channel-api)
    * [Type definitions](#type-definitions)
    * [Channel lifecycle](#channel-lifecycle)
    * [Channel usage](#channel-usage)
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
- **[SEGMENTATION-API](/standards/application/segmentation-api.md)** to handle large messages exceeding network size limits.
- **[SDS](https://lip.logos.co/ift-ts/raw/sds.html)** to provide causal-history-based end-to-end acknowledgement and retransmission.
- **Rate Limit Manager** to comply with [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) constraints when sending segmented messages.
- **Encryption Hook** to allow upper layers to provide a pluggable encryption mechanism. The main motivation for encryption is to protect the payload and content_topic from MessageEnvelope, defined in [MESSAGING-API](/standards/application/messaging-api.md). 

The separation between Reliable Channels and encryption ensures the API remains agnostic to identity and key management concerns,
which are handled by higher layers.

## Syntax

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## API design

### IDL

A custom Interface Definition Language (IDL) in YAML is used, consistent with [MESSAGING-API](/standards/application/messaging-api.md).

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

## The Reliable Channel API

This API considers the types defined by [MESSAGING-API](/standards/application/messaging-api.md) plus the following.

### Type definitions

```yaml
types:
  ReliableSendId:
    type: string
    description: "Unique identifier for a single `send` operation on a reliable channel.
    It groups all chunks produced by segmenting one envelope, so callers can correlate
    acknowledgement and error events back to the original send call.
    Internally, each chunk is dispatched as an independent [MESSAGING-API](/standards/application/messaging-api.md) call,
    producing one `RequestId` (as defined in [MESSAGING-API](/standards/application/messaging-api.md)) per chunk.
    A single `ReliableSendId` therefore maps to one or more underlying `RequestId` values,
    one per chunk sent."

  IEncryption:
    type: object
    description: "Interface for a pluggable encryption mechanism.
    When provided, see ReliableChannelConfig, the API consumer MUST implement both encrypt and decrypt operations.
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

  IPersistence:
    type: object
    description: "Interface for a pluggable SDS persistence backend.
    Implementations MUST provide all functions required to save and retrieve SDS state per channel. Implementations MUST also provide the persistence method of interest, e.g., SQLite, custom encrypted storage, etc.
    Refer to the [SDS spec](https://lip.logos.co/ift-ts/raw/sds.html) for the full definition of what state must be persisted."

  ReliableEnvelope:
    type: object
    description: "Wraps a MessageEnvelope (defined in [MESSAGING-API](/standards/application/messaging-api.md)) with a reliable channel identifier.
    An empty channelId marks the message as ephemeral; a non-empty value routes it through the named reliable channel for segmentation, SDS tracking, and encryption."
    fields:
      messageEnvelope:
        type: MessageEnvelope
        description: "The message payload and metadata. Refer to [MESSAGING-API](/standards/application/messaging-api.md) for details."
      channelId:
        type: string
        default: ""
        description: "Reliable channel identifier.
        If empty, the message is ephemeral: not tracked by SDS, never retransmitted, and NEVER segmented.
        Ephemeral payloads exceeding the network size limit MUST be rejected with an error.
        When the rate limit is approached, ephemeral messages are dropped immediately rather than queued.
        If non-empty, the messageEnvelope is segmented, registered with SDS, and encrypted under the named reliable channel."

  ReliableChannel:
    type: object
    description: "An entity that guarantees message delivery among all participants on a single content topic."
    fields:
      channelId:
        type: string
        description: "Unique identifier"
      messageEvents:
        type: MessageEvents
        description: "Event emitter for message-related events on this channel"

  ReliableChannelConfig:
    type: object
    fields:
      segmentationConfig:
        type: SegmentationConfig
        description: "Configuration for message segmentation. Refer to [SEGMENTATION-API](./segmentation-api.md) for details."
      sdsConfig:
        type: SdsConfig
        description: "Configuration for Scalable Data Sync."
      rateLimitConfig:
        type: RateLimitConfig
        default: DefaultRateLimitConfig
        description: "Configuration for rate limit management."
      encryption:
        type: optional<IEncryption>
        default: none
        description: "Optional pluggable encryption implementation. If none, messages are sent unencrypted."

  SdsConfig:
    type: object
    fields:
      persistence:
        type: IPersistence
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
    fields:
      enabled:
        type: bool
        default: true
        description: "Whether rate limiting is enforced. SHOULD be true when RLN is active."
      epochSizeMs:
        type: uint
        default: 600000  # 10 minutes
        description: "The epoch size used by the RLN relay, in milliseconds."

  MessageReceivedEvent:
    type: object
    description: "Event emitted when a complete message has been received and reassembled."
    fields:
      requestId:
        type: ReliableSendId
        description: "Identifier of the `send` operation that produced this message."
      envelope:
        type: ReliableEnvelope
        description: "The reassembled message and its channel context. `envelope.messageEnvelope.payload` contains the fully reassembled content; `envelope.channelId` identifies the channel on which it arrived."

  MessageSentEvent:
    type: object
    description: "Event emitted when all chunks of a message have been acknowledged by the network."
    fields:
      requestId:
        type: ReliableSendId
        description: "The identifier of the `send` operation whose chunks have all been acknowledged."

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

  MessageEvents:
    type: event_emitter
    description: "Event source for reliable message events on a channel"
    events:
      "reliable:message:received":
        type: MessageReceivedEvent
      "reliable:message:sent":
        type: MessageSentEvent
      "reliable:message:send-error":
        type: MessageSendErrorEvent
```

### Channel lifecycle

```yaml
functions:

  createReliableChannel:
    description: "Opens a reliable channel. Sets up the required SDS state, segmentation, and encryption."
    parameters:
      - name: channelConfig
        type: ReliableChannelConfig
        description: "Configuration for the channel."
      - name: senderId
        type: string
        description: "An identifier for this sender. SHOULD be unique and persisted between sessions."
    returns:
      type: result<ReliableChannel, error>

  closeChannel:
    description: "Closes a reliable channel and releases all associated resources and internal state."
    parameters:
      - name: channel
        type: ReliableChannel
        description: "The channel to close."
    returns:
      type: result<void, error>
```

**State management**:

Each `ReliableChannel` MUST maintain internal state for [SDS](https://lip.logos.co/ift-ts/raw/sds.html). This state MAY be persisted within SDS implementation layer and includes:
- A committed message log recording sent messages and their acknowledgement status.
- An outgoing buffer of unacknowledged message chunks pending confirmation.
- An incoming buffer for partially received segmented messages awaiting full reassembly.

The state MUST be persisted using the `persistence` backend specified in `SdsConfig`.
The default `memory` backend does not survive process restarts; implementors providing persistence SHOULD use a durable backend.

**Encryption**:

The `encryption` field in `ReliableChannelConfig` is intentionally optional.
The Reliable Channel API is agnostic to encryption mechanisms.

When an `IEncryption` implementation is provided, it MUST be applied as described in [Channel usage](#channel-usage).

### Channel usage

```yaml
functions:
  send:
    description: "Send a message through the reliable channel API. Routing and guarantees are determined by the envelope's channelId: empty means ephemeral; non-empty applies segmentation, SDS, and encryption (if configured) before dispatching."
    parameters:
      - name: envelope
        type: ReliableEnvelope
        description: "The envelope containing the message and channel routing information."
    returns:
      type: result<ReliableSendId, error>
      description: "Returns a `ReliableSendId` that callers can use to correlate subsequent `MessageSentEvent` or `MessageSendErrorEvent` events."
```

Incoming events are emitted on `ReliableChannel.messageEvents` as defined by `MessageEvents`.

**Outgoing message processing order**:

When `send` is called with a `ReliableEnvelope` whose `channelId` is non-empty, the implementation MUST process `envelope.messageEnvelope.payload` in the following order:

1. **Segment**: Split the payload into chunks as defined in [SEGMENTATION-API](./segmentation-api.md).
2. **Apply [SDS](https://lip.logos.co/ift-ts/raw/sds.html)**: Register each chunk with the SDS layer to track acknowledgements and enable retransmission.
3. **Encrypt**: If an `IEncryption` implementation is provided, encrypt each chunk before transmission.
4. **Rate Limit**: If `RateLimitConfig.enabled` is `true`, delay dispatch as needed to comply with [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) epoch constraints.
5. **Dispatch**: Send each chunk via the underlying [MESSAGING-API](/standards/application/messaging-api.md).

**Incoming message processing order**:

When a chunk is received from the network, the implementation MUST process it in the following order:

1. **Decrypt**: If an `IEncryption` implementation is provided, decrypt the chunk.
2. **Apply [SDS](https://lip.logos.co/ift-ts/raw/sds.html)**: Deliver the chunk to the SDS layer, which emits acknowledgements and detects gaps.
3. **Reassemble**: Once all chunks for a message have been received, reassemble and emit a `reliable:message:received` event.

**Retransmission**:

A certain Reliable Channel MUST retransmit unacknowledged chunks after `acknowledgementTimeoutMs`,
according to the [SDS](https://lip.logos.co/ift-ts/raw/sds.html) channel state,
up to `maxRetransmissions` times.
If a chunk is not acknowledged after all retransmission attempts, a `reliable:message:send-error` event MUST be emitted.

**Rate limiting**:

When `RateLimitConfig.enabled` is `true`, the implementation MUST space chunk transmissions
to comply with the [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) epoch constraints defined in `epochSizeMs`.
Chunks MUST NOT be sent at a rate that would violate the RLN message rate limit for the active epoch.

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
- The `IEncryption` interface MUST be implemented by the caller.
- The Reliable Channel API MUST NOT impose any specific encryption scheme.

## Security/Privacy Considerations

- This API does not provide confidentiality by default. An `IEncryption` implementation MUST be supplied when confidentiality is required.
- Chunk metadata (message ID, chunk index, total chunks) is visible to network observers unless encrypted by the hook.
- SDS acknowledgement messages are sent over the same content topic and are subject to the same confidentiality concerns.
- Rate limiting compliance is required to avoid exclusion from the network by RLN-enforcing relays.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
