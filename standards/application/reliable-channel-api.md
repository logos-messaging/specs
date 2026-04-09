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
    * [Common](#common)
      * [Common type definitions](#common-type-definitions)
    * [Channel](#channel)
      * [Channel type definitions](#channel-type-definitions)
      * [Channel function definitions](#channel-function-definitions)
      * [Channel predefined values](#channel-predefined-values)
      * [Channel extended definitions](#channel-extended-definitions)
    * [Messaging](#messaging)
      * [Messaging type definitions](#messaging-type-definitions)
      * [Messaging function definitions](#messaging-function-definitions)
      * [Messaging extended definitions](#messaging-extended-definitions)
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
an application-level interface that sits between the Logos Chat and the [MESSAGING-API](/standards/application/messaging-api.md) plus [P2P-RELIABILITY](/standards/application/p2p-reliability.md), i.e., `logos-chat` <-> **reliable-channel-api** <-> `messaging-api/p2p-reliability`.

It bundles segmentation, end-to-end reliability via [Scalable Data Sync (SDS)](https://lip.logos.co/ift-ts/raw/sds.html), rate limit management, and a pluggable encryption hook
into a single interface for sending and receiving messages reliably.


## Motivation

The [MESSAGING-API](/standards/application/messaging-api.md) provides peer-to-peer reliability via [P2P-RELIABILITY](/standards/application/p2p-reliability.md),
but does not provide high end-to-end delivery guarantees from sender to recipient.

This API addresses that gap by introducing:
- **Segmentation** to handle large messages exceeding network size limits.
- **SDS** to provide causal-history-based end-to-end acknowledgement and retransmission.
- **Rate Limit Manager** to comply with [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) constraints when sending segmented messages.
- **Encryption Hook** to allow upper layers (e.g., the Logos Chat) to provide a pluggable encryption mechanism.

The separation between Reliable Channels and encryption ensures the API remains agnostic to identity and key management concerns,
which are handled by higher layers.

## Syntax

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## API design

### IDL

A custom Interface Definition Language (IDL) in YAML is used, consistent with [MESSAGING-API](/standards/application/messaging-api.md).

### Architectural position

The Reliable Channel API sits between the Logos Chat and the Messaging API, as follows:

```
┌────────────────────────────────────────────────────────────┐
│                      Logos Chat                            │
│              (Identity + Encryption + UX)                  │
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

### Common

#### Common type definitions

This API considers the types defined by [MESSAGING-API](/standards/application/messaging-api.md) plus the following:

```yaml
types:
  MessageChunk:
    type: object
    description: "Represents the minimum unit of data transmitted across the network. Its size is governed by SegmentationConfig. Chunks are chained to their predecessors, and any chunk whose linkage to its predecessor cannot be verified MUST be discarded.
    Only the first chunk on every message, is rate-limited (RLN) and the subsequent chunks MUST be accepted rate-limit wise."
    fields:
      chunkId:
        type: string
        description: UUID for the chunk
      previousChunkId:
        type: string
        description: Previous chunk ID. Chunks with unvalidated previous chunks MUST be rejected.
        This field is empty when chunkIndex == 0.
      chunkIndex:
        type: uint
        description: "Zero-based position of this chunk within the ordered sequence of chunks that form the original message."
      lastChunkIndex:
        type: uint
        description: "All chunks of same message have the same value, i.e., num-chunks - 1."
      requestId:
        type: RequestId
        description: "The request id this chunk belongs to. This is generated by the send function and its type is defined in messaging-api."

  IEncryption:
    type: object
    description: "Interface for a pluggable encryption mechanism.
    When IEncryption is provided, see ReliableChannelConfig, the encrypt and decrypt operations MUST be provided by the API consumer.
    Notice that the encrypt/decrypt signatures may defer from what are described below but the important
    point is that both operations MUST have a mechanism to accept an byte array as input and return a
    byte array as an output."
    fields:
      encrypt:
        type: function
        description: "Encrypts a byte payload. Returns the encrypted payload."
        parameters:
          - name: plaintext
            type: array<byte>
        returns:
          type: result<array<byte>, error>
      decrypt:
        type: function
        description: "Decrypts a byte payload. Returns the decrypted payload."
        parameters:
          - name: ciphertext
            type: array<byte>
        returns:
          type: result<array<byte>, error>
```

### Channel

#### Channel type definitions

```yaml
types:
  ReliableEnvelope:
    type: object
    description: "Represents an extension of the MessageEnvelope defined in MESSAGING-API aiming mostly to allow
    including a reliable channel id attribute that, when given, it will assume the message should be sent reliably. Therefore, we leave open for implementors to replace the MessageEnvelope implementation and just add the reliable channel id there."
    fields:
      messageEnvelope:
        type: MessageEnvelope
        description: "Please refer to the [MESSAGING-API](/standards/application/messaging-api.md) for details."
      channelId:
        type: string
        default: ""
        description: "Reliable channel identifier.
        If empty, the message is considered ephemeral and is not sent reliably.
        If != empty, the messageEnvelope will be segmented, SDS'ed and encrypted under the given reliable channel."

  ReliableChannel:
    type: object
    description: "An entity that guarantees message delivery among all participants on a single content topic."
    fields:
      channelId:
        type: string
        description: "Unique identifier"
      messageEvents:
        type: ReliableMessageEvents
        description: "Event emitter for message-related events on this channel"

  ReliableChannelConfig:
    type: object
    fields:
      segmentationConfig:
        type: SegmentationConfig
        default: DefaultSegmentationConfig
        description: "Configuration for message segmentation."
      sdsConfig:
        type: SdsConfig
        default: DefaultSdsConfig
        description: "Configuration for Scalable Data Sync."
      rateLimitConfig:
        type: RateLimitConfig
        default: DefaultRateLimitConfig
        description: "Configuration for rate limit management."
      encryption:
        type: optional<IEncryption>
        default: none
        description: "Optional pluggable encryption implementation. If none, messages are sent unencrypted."

  SegmentationConfig:
    type: object
    fields:
      chunkSizeBytes:
        type: uint
        default: 102400  # 100 KiB
        description: "Maximum chunk size in bytes.
        Messages larger than this value are split before SDS processing."

  SdsConfig:
    type: object
    fields:
      historyBackend:
        type: string
        default: "memory"
        description: "Backend for persisting the SDS local history. Implementations MAY support custom backends (e.g., 'memory', 'sqlite')."
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
        description: "The epoch size used by the RLN relay, in milliseconds. Only the first message chunk is rate limited."
```

#### Channel function definitions

```yaml
functions:

  openChannel:
    description: "Opens a reliable channel. Sets up the required SDS state, segmentation, and encryption."
    parameters:
      - name: nodeConfig
        type: NodeConfig
        description: "The node configuration. See [MESSAGING-API](/standards/application/messaging-api.md)."
      - name: channelConfig
        type: ReliableChannelConfig
        description: "Configuration for the channel."
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

#### Channel predefined values

```yaml
values:

  DefaultSegmentationConfig:
    type: SegmentationConfig
    fields:
      chunkSizeBytes: 102400  # 100 KiB

  DefaultSdsConfig:
    type: SdsConfig
    fields:
      historyBackend: "memory"
      acknowledgementTimeoutMs: 5000
      maxRetransmissions: 5
      causalHistorySize: 2

  DefaultRateLimitConfig:
    type: RateLimitConfig
    fields:
      enabled: true
      epochSizeMs: 600000
```

#### Channel extended definitions

**State management**:

Each `ReliableChannel` MUST maintain internal state for SDS, including:
- A committed message log recording sent messages and their acknowledgement status.
- An outgoing buffer of unacknowledged message chunks pending confirmation.
- An incoming buffer for partially received segmented messages awaiting full reassembly.

The state MUST be persisted according to the `historyBackend` specified in `SdsConfig`.
The default `memory` backend does not survive process restarts; implementors providing persistence SHOULD use a durable backend.

**Encryption**:

The `encryption` field in `ReliableChannelConfig` is intentionally optional.
The Reliable Channel API is agnostic to encryption mechanisms.

When an `IEncryption` implementation is provided, it MUST be applied as described in the [Messaging extended definitions](#messaging-extended-definitions).

### Messaging

#### Messaging type definitions

```yaml
types:
  ReliableMessageReceivedEvent:
    type: object
    description: "Event emitted when a complete message has been received and reassembled."
    fields:
      requestId:
        type: RequestId
        description: "Identifier of the `send` operation that initiated the message sending"
      contentTopic:
        type: string
        description: "Content topic on which the message was received"
      payload:
        type: array<byte>
        description: "The decrypted and reassembled message payload"

  ReliableMessageSentEvent:
    type: object
    description: "Event emitted when all chunks of a message have been acknowledged by the network."
    fields:
      requestId:
        type: RequestId
        description: "The request ID associated with the sent message"

  ReliableMessageSendErrorEvent:
    type: object
    description: "Event emitted when a message send operation fails after exhausting retransmission attempts."
    fields:
      requestId:
        type: RequestId
        description: "The request ID associated with the failed message"
      error:
        type: string
        description: "Error message describing what went wrong"

  ReliableMessageEvents:
    type: event_emitter
    description: "Event source for reliable message events on a channel"
    events:
      "reliable:message:received":
        type: ReliableMessageReceivedEvent
      "reliable:message:sent":
        type: ReliableMessageSentEvent
      "reliable:message:send-error":
        type: ReliableMessageSendErrorEvent
```

#### Messaging function definitions

```yaml
functions:
  send:
    description: "Send a message reliably through the channel. Applies segmentation, SDS, and encryption (if configured) before dispatching."
    parameters:
      - name: channel
        type: optional<ReliableChannel>
        description: "The reliable channel to use for sending. If not provided, the message is sent without delivery guarantees and is treated as ephemeral."
      - name: payload
        type: array<byte>
        description: "The raw message payload to send."
    returns:
      type: result<RequestId, error>
```

#### Messaging extended definitions

**Outgoing message processing order**:

When `send` is called, the implementation MUST process the message in the following order:

1. **Segment**: Split the payload into chunks of at most `chunkSizeBytes` as defined in `SegmentationConfig`.
2. **Apply SDS**: Register each chunk with the SDS layer to track acknowledgements and enable retransmission.
3. **Encrypt**: If an `IEncryption` implementation is provided, encrypt each chunk before transmission.
4. **Dispatch**: Send each chunk via the underlying [MESSAGING-API](/standards/application/messaging-api.md).

**Incoming message processing order**:

When a chunk is received from the network, the implementation MUST process it in the following order:

1. **Decrypt**: If an `IEncryption` implementation is provided, decrypt the chunk.
2. **Apply SDS**: Deliver the chunk to the SDS layer, which emits acknowledgements and detects gaps.
3. **Reassemble**: Once all chunks for a message have been received, reassemble and emit a `reliable:message:received` event.

**Retransmission**:

The SDS layer MUST retransmit unacknowledged chunks after `acknowledgementTimeoutMs`,
up to `maxRetransmissions` times.
If a chunk is not acknowledged after all retransmission attempts, a `reliable:message:send-error` event MUST be emitted.

**Rate limiting**:

When `RateLimitConfig.enabled` is `true`, the implementation MUST space chunk transmissions
to comply with the RLN epoch constraints defined in `epochSizeMs`.
Chunks MUST NOT be sent at a rate that would violate the RLN message rate limit for the active epoch.

## Components

### Segmentation

Segmentation splits outgoing messages into chunks before SDS processing.
This ensures each chunk fits within the network's maximum message size.

- The default chunk size is `100 KiB` (`102400 bytes`.)
- Chunks MUST be tagged with metadata sufficient for the receiver to reassemble the original message:
  a message identifier, chunk index, and total chunk count.
- Segmentation MUST be transparent to the SDS layer; SDS operates on individual chunks.
- The maximum allowed message size is `1MiB` (`1048576 bytes`.)
- Messages that are bigger than the maximum allowed size will be discarded automatically and an error will be given to the caller.

### Scalable Data Sync (SDS)

SDS provides end-to-end delivery guarantees using causal history tracking.

- Each sent chunk is registered in an outgoing buffer.
- The recipient sends acknowledgements back to the sender upon receiving chunks.
- The sender removes acknowledged chunks from the outgoing buffer.
- Unacknowledged chunks are retransmitted after `acknowledgementTimeoutMs`.
- SDS state MUST be persisted in the configured `historyBackend`.

### Rate Limit Manager

The Rate Limit Manager ensures compliance with [RLN](https://lip.logos.co/messaging/standards/core/17/rln-relay.html) rate constraints.

- It tracks how many messages have been sent in the current epoch (only the first chunk of each message counts toward the rate limit; subsequent chunks are exempt).
- When the limit is approached, chunk dispatch MUST be delayed to the next epoch.
- The epoch size MUST match the `epochSizeMs` configured in `RateLimitConfig`.

### Encryption Hook

The Encryption Hook provides a pluggable interface for upper layers to inject encryption.

- The hook is optional; when not provided, messages are sent as plaintext.
- Encryption is applied per chunk, after segmentation and SDS registration.
- Decryption is applied per chunk, before SDS delivery.
- The `IEncryption` interface MUST be implemented by the caller (e.g., the Logos Chat).
- The Reliable Channel API MUST NOT impose any specific encryption scheme.

## Security/Privacy Considerations

- This API does not provide confidentiality by default. An `IEncryption` implementation MUST be supplied when confidentiality is required.
- Chunk metadata (message ID, chunk index, total chunks) is visible to network observers unless encrypted by the hook.
- SDS acknowledgement messages are sent over the same content topic and are subject to the same confidentiality concerns.
- Rate limiting compliance is required to avoid exclusion from the network by RLN-enforcing relays.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
