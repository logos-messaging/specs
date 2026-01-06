---
title: RELIABLE-CHANNEL-API
name: Reliable Channel API
category: Standards Track
status: raw
tags: [reliability, application, api]
editor: Franck Royer <franck@status.im>
---

## Table of contents

<!-- TOC -->
  * [Table of contents](#table-of-contents)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
  * [Syntax](#syntax)
  * [API design](#api-design)
    * [IDL](#idl)
    * [Primitive types and general guidelines](#primitive-types-and-general-guidelines)
  * [Architecture](#architecture)
    * [SDS integration](#sds-integration)
    * [Message segmentation](#message-segmentation)
    * [Rate limit management](#rate-limit-management)
  * [The Reliable Channel API](#the-reliable-channel-api)
    * [Create reliable channel](#create-reliable-channel)
      * [Type definitions](#type-definitions)
      * [Function definitions](#function-definitions)
    * [Send messages](#send-messages)
      * [Function definitions](#function-definitions-1)
    * [Event handling](#event-handling)
      * [Type definitions](#type-definitions-1)
      * [Extended definitions](#extended-definitions)
    * [Channel lifecycle](#channel-lifecycle)
      * [Function definitions](#function-definitions-2)
  * [Future improvements](#future-improvements)
  * [Security/Privacy considerations](#securityprivacy-considerations)
  * [Copyright](#copyright)
<!-- TOC -->

## Abstract

This document specifies an Application Programming Interface (API) for Reliable Channel,
a high-level abstraction that provides eventual message consistency guarantees for all participants in a channel,
as well as message segmentation and rate limit management when using an underlying rate limited delivery protocol
with message size restrictions such as [WAKU2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md).

The Reliable Channel is built on top of:
- [WAKU-API](/standards/application/waku-api.md) for Waku protocol integration
- [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) (Scalable Data Sync) for causal ordering and acknowledgements
- Message segmentation for handling large payloads
- Rate limit management for [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md) compliance

The Reliable Channel API ensures that:

- All messages sent in a channel are eventually received by all participants
- Senders are notified when messages are acknowledged by other participants
- Missing messages are automatically detected and retrieved
- Message delivery is retried until acknowledged or maximum retry attempts are reached
- Messages are causally ordered using Lamport timestamps
- Large messages can be segmented to fit transport constraints
- Messages are queued or dropped when the underlying routing transport has a rate limit and said limit is being reached

## Motivation

While protocols like [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) provide the mechanisms for achieving reliability
(causal ordering, acknowledgements, missing message detection),
and [WAKU-API](/standards/application/waku-api.md) provides the transport layer,
there is a need for an opinionated, high-level API that makes these capabilities accessible and easy to use.

The Reliable Channel API provides this accessibility by:
- **Simplifying integration**: Wraps SDS, Waku protocols, and other components (segmentation, rate limiting) behind a single, cohesive interface
- **Providing sane defaults**: Pre-configures SDS parameters, retry strategies, and sync intervals for common use cases
- **Event-driven model**: Exposes message lifecycle through intuitive events rather than requiring manual polling of SDS state
- **Automatic task scheduling**: Handles the periodic execution of SDS tasks (sync, buffer sweeps) internally
- **Abstracting complexity**: Hides the details of:
  - SDS message wrapping/unwrapping
  - Store queries for missing messages
  - Message segmentation for large payloads
  - Rate limit compliance when using [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)

The goal is to enable application developers to achieve end-to-end reliability
with minimal configuration and without deep knowledge of the underlying protocols.
This follows the same philosophy as [WAKU-API](/standards/application/waku-api.md):
providing an opinionated, accessible interface to powerful but complex underlying mechanisms.

## Syntax

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

## API design

### IDL

This specification uses the same custom Interface Definition Language (IDL) in YAML as defined in [WAKU-API](/standards/application/waku-api.md).

### Primitive types and general guidelines

The primitive types and general guidelines are the same as defined in [WAKU-API](/standards/application/waku-api.md#primitive-types-and-general-guidelines).

## Architecture

The Reliable Channel is a layered architecture that combines multiple components:

```
┌──────────────────────────────────────────────────────────┐
│      Reliable Channel API                                │ ← Application-facing event-driven API
├──────────────────────────────────────────────────────────┤
│  Message Segmentation                                    │ ← Large message splitting/reassembly
├──────────────────────────────────────────────────────────┤
│  Rate Limit Manager                                      │ ← WAKU2-RLN-RELAY compliance & pacing
├──────────────────────────────────────────────────────────┤
│  SDS (Scalable Data Sync)                                │ ← Causal ordering & acknowledgements
├──────────────────────────────────────────────────────────┤
│  WAKU-API (LightPush/Filter/Store/P2P Reliability)       │ ← Message transport layer
└──────────────────────────────────────────────────────────┘
```

### SDS integration

The Reliable Channel wraps the [SDS](https://github.com/vacp2p/rfc-index/blob/main/vac/raw/sds.md) `MessageChannel`, which provides:

- **Causal ordering**: Using Lamport timestamps to establish message order
- **Acknowledgments**: Via causal history (definitive) and bloom filters (probabilistic)
- **Missing message detection**: By tracking gaps in causal history
- **Buffering**: For unacknowledged outgoing messages and incoming messages with unmet dependencies

The Reliable Channel handles the integration between SDS and Waku protocols:
- Wrapping user payloads in SDS messages before encoding
- Unwrapping SDS messages after decoding (extracting `content` field)
- Subscribing to messages via the Waku node's unified subscribe API
- Receiving messages via the node's message emitter (content-topic based)
- Scheduling SDS periodic tasks (sync, buffer sweeps, process tasks)
- Mapping SDS events to user-facing events
- Computing retrieval hints (Waku message hashes) for SDS messages

### Message segmentation

For messages exceeding safe payload limits:
- Messages SHOULD be split into segments of approximately 100 KB
- The 100 KB limit accounts for overhead from:
  - Encryption (message-encryption layer)
  - [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md) proof
  - SDS metadata (causal history, bloom filter, Lamport timestamp)
  - Protocol buffers encoding
- This ensures the final encoded Waku message stays well below the 150 KB routing layer limit
- Each segment SHOULD be tracked independently through SDS
- Segments SHOULD be reassembled before delivery to the application
- Partial message state SHOULD be managed to handle segment loss
- Segment order MUST be preserved during reassembly

TODO: refer to message segmentation spec

### Rate limit management

When using [WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md):
- Regular messages and segments exceeding the rate limit SHOULD be queued
- Ephemeral messages SHOULD be dropped (not sent nor queued) when the rate limit is approached or reached
- "Approached" threshold SHOULD be configurable (e.g., drop ephemerals at 90% capacity)
- Rate limit errors SHOULD be surfaced through events
- When segmentation is enabled, each segment counts toward the rate limit independently
- Messages coming from a retry mechanism should be queued when the rate limit is approached.

TODO: refer to rate limit manager spec

## The Reliable Channel API

```yaml
api_version: "0.0.1"
library_name: "waku-reliable-channel"
description: "Reliable Channel: an event-driven API for eventual message consistency in Waku channels."
```

### Create reliable channel

#### Type definitions

```yaml
types:
  ReliableChannel:
    type: object
    description: "A Reliable Channel instance that provides eventual consistency guarantees."
    fields:
      sync_status:
        type: SyncStatus
        description: "An event target that emits synchronization status events. Use this to track whether the channel is fully synchronized with all participants."

  ReliableChannelOptions:
    type: object
    fields:
      sync_min_interval_ms:
        type: uint
        default: 30000
        description: "The minimum interval between 2 sync messages in the channel (in milliseconds). This is shared responsibility between channel participants. Set to 0 to disable automatic sync messages."
      retry_interval_ms:
        type: uint
        default: 30000
        description: "How long to wait before re-sending a message that has not been acknowledged (in milliseconds)."
      max_retry_attempts:
        type: uint
        default: 10
        description: "How many times to attempt resending messages that were not acknowledged."
      retrieve_frequency_ms:
        type: uint
        default: 10000
        description: "How often store queries are done to retrieve missing messages (in milliseconds)."
      sweep_in_buf_interval_ms:
        type: uint
        default: 5000
        description: "How often the SDS message channel incoming buffer is swept (in milliseconds)."
      query_on_connect:
        type: bool
        default: true
        description: "Whether to automatically do a store query after connection to store nodes. Note: this should be moved to the Waku API."
      auto_start:
        type: bool
        default: true
        description: "Whether to automatically start the message channel."
      process_task_min_elapse_ms:
        type: uint
        default: 1000
        description: "The minimum elapsed time between calling the underlying SDS channel process task for incoming messages. This prevents overload when processing many messages."
      causal_history_size:
        type: uint
        default: 200
        description: "The number of recent messages to include in causal history. Passed to the underlying SDS MessageChannel."
      timeout_for_lost_messages_ms:
        type: uint
        default: 0
        description: "The time in milliseconds after which a message dependency that could not be resolved is marked as irretrievable. Disabled if undefined or 0. Passed to the underlying SDS MessageChannel."
      possible_acks_threshold:
        type: uint
        default: 2
        description: "How many possible acknowledgements (bloom filter hits) does it take to consider it a definitive acknowledgement. Passed to the underlying SDS MessageChannel."

  ChannelId:
    type: string
    description: "An identifier for the channel. All participants of the channel MUST use the same id."

  SenderId:
    type: string
    description: "An identifier for the sender. SHOULD be unique per participant and persisted between sessions to ensure acknowledgements are only valid when originating from different senders."

  MessageId:
    type: string
    description: "A unique identifier for a logical message, derived from the message payload before segmentation."

  MessagePayload:
    type: array<byte>
    description: "The unwrapped message content (user payload extracted from SDS message)."

  Encryption:
    type: object
    description: "Handles for encryption and decryption of messages. Encryption happens after SDS wrapping and before sending; decryption happens after receiving and before SDS unwrapping."
    fields:
      encrypt:
        type: function
        parameters:
          - name: clear_payload
            type: array<byte>
        returns: array<byte>
        description: "Encrypts the clear payload (SDS-wrapped message)."
      decrypt:
        type: function
        parameters:
          - name: encrypted_payload
            type: array<byte>
        returns: array<byte>
        description: "Decrypts the encrypted payload to reveal the SDS-wrapped message."
```

#### Function definitions

```yaml
functions:
  createReliableChannel:
    description: "Create a new Reliable Channel instance. All participants in the channel MUST be able to decrypt messages and MUST subscribe to the same content topic(s)."
    parameters:
      - name: waku_node
        type: WakuNode
        description: "The Waku node instance to use for sending and receiving messages."
      - name: channel_id
        type: ChannelId
        description: "An identifier for the channel. All participants MUST use the same id."
      - name: sender_id
        type: SenderId
        description: "An identifier for this sender. SHOULD be unique and persisted between sessions."
      - name: content_topic
        type: ContentTopic
        description: "The content topic to use for the channel."
      - name: encryption
        type: Encryption
        description: "Optional encryption/decryption interface. If not provided, messages are sent without encryption (passthrough)."
      - name: options
        type: ReliableChannelOptions
        description: "Configuration options for the Reliable Channel."
    returns:
      type: result<ReliableChannel, error>
```


### Send messages

#### Function definitions

```yaml
functions:
  send:
    description: "Send a message in the channel. The message will be retried if not acknowledged by other participants."
    parameters:
      - name: message_payload
        type: array<byte>
        description: "The message content to send (before SDS wrapping)."
    returns:
      type: MessageId
      description: "A unique identifier for the message, used to track events."

  sendEphemeral:
    description: "Send an ephemeral message in the channel. Ephemeral messages are not tracked for acknowledgement, not included in causal history, and are dropped when rate limits are approached or reached."
    parameters:
      - name: message_payload
        type: array<byte>
        description: "The message content to send (before SDS wrapping)."
    returns:
      type: MessageId
      description: "A unique identifier for the message, used to track events (limited to `message-sent` or `ephemeral-message-dropped` event)."
```

### Event handling

The Reliable Channel uses an event-driven model to notify applications about message lifecycle events.

#### Type definitions

```yaml
types:
  ChunkInfo:
    type: object
    description: "Information about message segmentation for tracking partial progress."
    fields:
      chunks:
        type: array<uint>
        description: "Array of chunk indices (e.g., [0, 1, 3] means chunks 0, 1, and 3). For non-segmented messages, this is [0]."
      total_chunks:
        type: uint
        description: "Total number of chunks for this message. For non-segmented messages, this is 1. When chunks.length == total_chunks, all chunks are complete."

  ReliableChannelEvents:
    type: object
    description: "Events emitted by the Reliable Channel."
    events:
      sending-message:
        type: event<SendingMessageEvent>
        description: "Emitted when a message chunk is being sent over the wire. MAY be emitted multiple times if retry mechanism kicks in. For segmented messages, emitted once per chunk."

      message-sent:
        type: event<MessageSentEvent>
        description: "Emitted when a message chunk has been sent over the wire but has not been acknowledged yet. MAY be emitted multiple times if retry mechanism kicks in. For segmented messages, emitted once per chunk."

      message-possibly-acknowledged:
        type: event<PossibleAcknowledgment>
        description: "Emitted when a bloom filter indicates a message chunk was possibly received by another party. This is probabilistic. For segmented messages, emitted per chunk."

      message-acknowledged:
        type: event<MessageAcknowledgedEvent>
        description: "Emitted when a message chunk was fully acknowledged by other members of the channel (present in their causal history). For segmented messages, emitted per chunk as each is acknowledged."

      sending-message-irrecoverable-error:
        type: event<MessageError>
        description: "Emitted when a message chunk could not be sent due to a non-recoverable error. For segmented messages, emitted per chunk that fails."

      message-received:
        type: event<MessagePayload>
        description: "Emitted when a new complete message has been received and reassembled from another participant. The payload is the unwrapped user content (SDS content field). Only emitted once all chunks are received."

      irretrievable-message:
        type: event<HistoryEntry>
        description: "Emitted when the channel is aware of a missing message but failed to retrieve it successfully."

      ephemeral-message-dropped:
        type: event<EphemeralDropReason>
        description: "Emitted when an ephemeral message was dropped due to rate limit constraints."

  SendingMessageEvent:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices being sent. The chunks array contains all chunks sending so far."

  MessageSentEvent:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have been sent. The chunks array contains all chunks sent so far."

  MessageAcknowledgedEvent:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have been acknowledged. The chunks array contains all acknowledged chunks so far. When chunks.length == total_chunks, all chunks are acknowledged."

  PossibleAcknowledgment:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID that was possibly acknowledged."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have been possibly acknowledged (via bloom filters). The chunks array contains all possibly acknowledged chunks so far. Note: this may not be an ideal API to use, further revision may be needed."

  MessageError:
    type: object
    fields:
      message_id:
        type: MessageId
        description: "The logical message ID that encountered an error."
      chunk_info:
        type: ChunkInfo
        description: "Chunk indices that have failed. The chunks array contains all failed chunks so far."
      error:
        type: error
        description: "The error that occurred."

  HistoryEntry:
    type: object
    description: "An entry in the message history that could not be retrieved."

  EphemeralDropReason:
    type: object
    fields:
      reason:
        type: string
        description: "The reason the ephemeral message was dropped (e.g., 'rate_limit_approached', 'rate_limit_reached')."

  SyncStatus:
    type: object
    description: "An event target that emits synchronization status events. Accessible via the reliable channel's sync_status property."
    events:
      synced:
        type: event<SyncStatusDetail>
        description: "Emitted when the channel is not aware of any missing messages that can still be retrieved. Note that some messages may have been permanently lost."

      syncing:
        type: event<SyncStatusDetail>
        description: "Emitted when the channel is aware of missing messages and is attempting to retrieve them."

  SyncStatusDetail:
    type: object
    description: "Details about the current synchronization state of the channel."
    fields:
      received:
        type: uint
        description: "Count of messages successfully obtained."
      missing:
        type: uint
        description: "Count of messages absent but potentially retrievable."
      lost:
        type: uint
        description: "Count of messages considered permanently lost (irretrievable due to timeout or unavailability)."
```

#### Extended definitions

**Event lifecycle**:

For each regular message sent via `send()`, the following event sequence is expected:

1. `sending-message`: Emitted each time chunks are sent, with cumulative array (MAY be emitted multiple times if retry mechanism kicks in)
   - Example progression for a 1-chunk message:
     - sending: `chunk_info={chunks: [0], total_chunks: 1}`
   - Example progression for a 5-chunk message:
     - First send: `chunk_info={chunks: [0], total_chunks: 5}`
     - Second send: `chunk_info={chunks: [0, 1], total_chunks: 5}`
     - Fifth send: `chunk_info={chunks: [0, 1, 2, 3, 4], total_chunks: 5}`
2. `message-sent`: Emitted with same cumulative pattern as `sending-message` (MAY be emitted multiple times if retry mechanism kicks in)
3. `message-acknowledged`: Emitted each time new chunks are acknowledged
   - `chunk_info.chunks` array grows as chunks are acknowledged
   - Example progression for a 1-chunk message:
     - Only ack: `chunk_info={chunks: [0], total_chunks: 1}` (complete when `chunks.length == total_chunks`)
   - Example progression for a 5-chunk message:
     - First ack: `chunk_info={chunks: [0], total_chunks: 5}`
     - Second ack: `chunk_info={chunks: [0, 1], total_chunks: 5}`
     - Third ack: `chunk_info={chunks: [0, 1, 3], total_chunks: 5}` (chunk 2 still pending)
     - Fourth ack: `chunk_info={chunks: [0, 1, 2, 3], total_chunks: 5}` (chunk 2 now received)
     - Final ack: `chunk_info={chunks: [0, 1, 2, 3, 4], total_chunks: 5}` (complete when `chunks.length == total_chunks`)
   - All events share the same `message_id`
   - Application can:
     - Check overall progress: `chunk_info.chunks.length / chunk_info.total_chunks` (e.g., 3/5 = 60%)
     - Check if complete: `chunk_info.chunks.length == chunk_info.total_chunks`
     - Track specific chunks: `chunk_info.chunks.includes(2)` to see if chunk 2 is done
     - Display which chunks remain: chunks not in the array

Alternatively, instead of `message-acknowledged`, the message lifecycle MAY end with:
   - `sending-message-irrecoverable-error`: Emitted if an unrecoverable error occurs

Optionally, before final acknowledgement:
   - `message-possibly-acknowledged`: (Probabilistic) MAY be emitted as bloom filter hits are detected

**For received messages**:
- `message-received`: Emitted **only once** after all chunks are received and reassembled.
- The payload is the complete reassembled message
- No chunk information is provided (reassembly is transparent to the receiver)

For ephemeral messages sent via `sendEphemeral()`:
- Ephemeral messages are NEVER segmented (if too large, they are rejected)
  Rationale: Segmenting ephemeral messages would consume multiple rate limit slots without guarantees that all chunks get across, potentially wasting bandwidth on incomplete messages that provide no value to the application
- `message-sent`: Emitted once with `chunk_info={chunks: [0], total_chunks: 1}`
- `ephemeral-message-dropped`: Emitted if the message is dropped due to rate limit constraints
- `sending-message-irrecoverable-error`: Emitted if encoding, sending, or size check fails

**Irrecoverable errors**:

The following errors are considered irrecoverable and will trigger `sending-message-irrecoverable-error`:
- Encoding failed
- Empty payload
- Message size too large
- WAKU2-RLN-RELAY proof generation failed

When an irrecoverable error occurs, the retry mechanism SHOULD NOT attempt to resend the message.

**Sync status**:

The `sync_status` property provides visibility into the channel's synchronization state.
This is useful for displaying loading indicators, alerting users about missing content, or detecting permanently lost data.

The channel operates in two states:

1. **Synced**: The channel is not aware of any missing messages that can still be retrieved.
   However, some data may have been permanently lost (indicated by `lost` count > 0).

2. **Syncing**: The channel is aware of missing messages and is attempting to retrieve them.

Events are emitted on the `sync_status` object (not the main channel):
- `synced`: Emitted when the channel transitions to the synced state
- `syncing`: Emitted when the channel transitions to the syncing state, or when the sync status details change while syncing

Both events include a `SyncStatusDetail` object with:
- `received`: Count of messages successfully obtained
- `missing`: Count of messages absent but potentially retrievable
- `lost`: Count of messages considered permanently lost

Lost messages are those that could not be retrieved within the configured `timeout_for_lost_messages_ms`.
When messages are marked as permanently lost, there is currently no automatic recovery mechanism available.
Future improvements MAY introduce additional recovery options.

### Channel lifecycle

#### Function definitions

```yaml
functions:
  start:
    description: "Start the Reliable Channel. Sets up event listeners, begins sync loop, starts missing message retrieval, and subscribes to messages via the Waku node."
    returns:
      type: void

  stop:
    description: "Stop the Reliable Channel. Stops sync loop, missing message retrieval, and clears intervals."
    returns:
      type: void
```

## Future improvements

- Developer may prefer to have control over the content topics for privacy and anonymity purposes, future improvements may enable this.
- Developers may prefer the message routing service (aka [WAKU2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)) to provide message rate limit information.
- "Query on connect" is most likely more suited at the [WAKU-API](/standards/application/waku-api.md) layer
- Message received events could be emitted per chunk, to enable the consumer to provide feedback to the users in terms of large messages being received.

## Security/Privacy considerations

1. **Encryption**: All participants in a Reliable Channel MUST be able to decrypt messages.
   Implementations SHOULD use the same encryption mechanisms for all messages.

2. **Sender identity**: The `sender_id` is used to differentiate acknowledgements.
   Implementations SHOULD ensure that acknowledgements are only considered valid when they originate from a different sender.

3. **Channel isolation**: Messages in different channels are isolated.
   A participant SHOULD only process messages that match their channel ID.

4. **Message ordering**: While the Reliable Channel ensures eventual consistency,
   it does not guarantee strict message ordering across participants.

5. **Resource exhaustion**: Implementations SHOULD implement limits on:
   - Number of missing messages tracked
   - Number of active retry attempts
   - Frequency of store queries

6. **Privacy**: Store queries reveal message interest patterns.
   Implementations MAY consider privacy-preserving retrieval strategies in the future.

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
