---
title: Message Segmentation and Reconstruction
name: Message Segmentation and Reconstruction
tags: [segmentation]
version: 0.1
status: raw
---

## Abstract

This specification defines an application-layer protocol for **segmentation** and **reconstruction** of messages carried over a transport/delivery service with a message-size limitation, when the original payload exceeds said limitation.
Applications partition the payload into multiple transport messages and reconstruct the original on receipt,
even when segments arrive out of order or up to a **predefined percentage** of segments are lost.
The protocol optionally uses **Reed–Solomon** erasure coding for fault tolerance.
All messages are wrapped in a `SegmentMessageProto`, including those that fit in a single segment.

## Motivation

Many message transport and delivery protocols impose a maximum message size that restricts the size of application payloads.
For example, Waku Relay typically propagates messages up to **150 KB** as per [64/WAKU2-NETWORK - Message](https://rfc.vac.dev/waku/standards/core/64/network#message-size).
To support larger application payloads, a segmentation layer is required.
This specification enables larger messages by partitioning them into multiple envelopes and reconstructing them at the receiver.
Erasure-coded parity segments provide resilience against partial loss or reordering.

## Terminology

- **original payload**: the full application payload before segmentation.
- **data segment**: one of the partitioned chunks of the original message payload.
- **parity segment**: an erasure-coded segment derived from the set of data segments.
- **segment message**: a wire-message whose `payload` field carries a serialized `SegmentMessageProto`.
- **`segmentSize`**: configured maximum size in bytes of each data segment's `payload` chunk (before protobuf serialization).

The key words **"MUST"**, **"MUST NOT"**, **"REQUIRED"**, **"SHALL"**, **"SHALL NOT"**, **"SHOULD"**, **"SHOULD NOT"**, **"RECOMMENDED"**, **"NOT RECOMMENDED"**, **"MAY"**, and **"OPTIONAL"** in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Wire Format

Each segmented message is encoded as a `SegmentMessageProto` protobuf message:

```protobuf
syntax = "proto3";

message SegmentMessageProto {
  // Keccak256(original payload), 32 bytes
  bytes  entire_message_hash    = 1;

  // Data segment indexing
  uint32 data_segment_index     = 2; // zero-indexed sequence number for data segments
  uint32 data_segment_count         = 3; // number of data segments (>= 1)

  // Segment payload (data or parity shard)
  bytes  payload                = 4;

  // Parity segment indexing
  uint32 parity_segment_index   = 5; // zero-based sequence number for parity segments
  uint32 parity_segment_count  = 6; // number of parity segments

  // Segment type
  bool   is_parity              = 7; // true for parity segments, false (default) for data segments
}
```

**Field descriptions:**

- `entire_message_hash`: A 32-byte Keccak256 hash of the original complete payload, used to identify which segments belong together and verify reconstruction integrity.
- `data_segment_index`: Zero-indexed sequence number identifying this data segment's position (0, 1, 2, ..., data_segment_count - 1). Set only on data segments.
- `data_segment_count`: Total number of data segments the original message was split into. Set on every segment (data and parity).
- `payload`: The actual chunk of data or parity information for this segment.
- `parity_segment_index`: Zero-based sequence number for parity segments. Set only on parity segments.
- `parity_segment_count`: Total number of parity segments generated. Set on every segment (data and parity) when Reed–Solomon parity is used; `0` (default) otherwise.
- `is_parity`: Explicit segment type marker. `false` (default) for data segments; `true` for parity segments.

A message is either a **data segment** (when `is_parity == false`) or a **parity segment** (when `is_parity == true`).

### Validation

Receivers **MUST** enforce:

- `entire_message_hash.length == 32`
- `data_segment_count >= 1`
- `data_segment_count + parity_segment_count < maxTotalSegments` 
- **Data segments** (`is_parity == false`):
  `data_segment_index < data_segment_count`
- **Parity segments** (`is_parity == true`):
  `parity_segment_count > 0` AND `parity_segment_index < parity_segment_count`

No other combinations are permitted.
A `SegmentMessageProto` with `data_segment_count == 1` and `data_segment_index == 0` is a valid single-segment data message: the `payload` field carries the entire original payload (see [Sending](#sending)).

## Segmentation

### Sending

To transmit a payload, the sender:

- **MUST** compute a 32-byte `entire_message_hash = Keccak256(original_payload)`.
- **MUST** split the payload into one or more **data segments**,
  each of size up to `segmentSize` bytes.
  A payload of size ≤ `segmentSize` produces a single data segment (`data_segment_count == 1`).
- **MUST** pad the last segment to `segmentSize` for Reed-Solomon erasure coding (only if Reed-Solomon coding is enabled)
- **MAY** use Reed–Solomon erasure coding at the predefined parity rate.
- **MUST** encode every segment as a `SegmentMessageProto` with:
  - The `entire_message_hash`
  - `data_segment_count` (total number of data segments, always set)
  - When Reed–Solomon parity is used, `parity_segment_count` (total number of parity segments, set on every segment)
  - For data segments: `is_parity = false`, `data_segment_index`
  - For parity segments: `is_parity = true`, `parity_segment_index`
  - The raw payload data
- Send each segment as an individual transport message according to the underlying transport service.

This yields a deterministic wire format: every transmitted payload is a `SegmentMessageProto`.

### Receiving

Upon receiving a segmented message, the receiver:

- **MUST** validate each segment according to [Wire Format → Validation](#validation).
- **MUST** cache received segments
- **MUST** attempt reconstruction once at least `data_segment_count` distinct segments (data and parity combined) have been received:
  - If all data segments are present, concatenate their `payload` fields in `data_segment_index` order.
  - Otherwise, recover the payload via Reed–Solomon decoding over the available data and parity segments.
- **MUST** verify `Keccak256(reconstructed_payload)` matches `entire_message_hash`.
  On mismatch,
  the message **MUST** be discarded and logged as invalid.
- Once verified,
  the reconstructed payload **SHALL** be delivered to the application.

---

## Implementation Suggestions

### Reed–Solomon

Implementations that apply parity **SHALL** use fixed-size shards of length `segmentSize`.
The reference implementation uses **nim-leopard** (Leopard-RS) with a maximum of **256 total shards**.

### Storage / Persistence

Segments may be persisted (e.g., SQLite) and indexed by `entire_message_hash` and by sender. Sender may be authenticated, this is out of scope of this spec.
Implementations **SHOULD** support:

- Duplicate detection and idempotent saves
- Completion flags to prevent duplicate processing
- Timeout-based cleanup of incomplete reconstructions
- Per-sender quotas for stored bytes and concurrent reconstructions

### Configuration

- `segmentSize` — maximum size in bytes of each data segment's payload chunk (before protobuf serialization).
    **REQUIRED** parameter, configurable by the client.
- `parityRate` — fraction of parity shards relative to data shards.
    Configurable by the client. Defaults to **0.125** (12.5%).
- `maxTotalSegments` — maximum number of total shards (data + parity) per message.
    Implementation-specific parameter, fixed. The reference implementation uses **256**.

**Reconstruction capability:**
With the predefined parity rate, reconstruction is possible if **all data segments** are received or if **any combination of data + parity** totals at least `data_segment_count` (i.e., up to the predefined percentage of loss tolerated).

**API simplicity:**
Libraries **SHOULD** require only `segmentSize` from the application for normal operation.

---

## Security Considerations

### Privacy

`entire_message_hash` enables correlation of segments that belong to the same original message but does not reveal content.
To prevent this correlation, applications **SHOULD** encrypt each segment after segmentation (see [Encryption](#encryption)).
Traffic analysis may still identify segmented flows.

### Encryption

This specification does not provide confidentiality.
Applications **SHOULD** encrypt each segment after segmentation
(i.e., encrypt the serialized `SegmentMessageProto` prior to transmission),
so that `entire_message_hash` and other identifying fields are not visible to observers.

### Integrity

Implementations **MUST** verify the Keccak256 hash post-reconstruction and discard on mismatch.

### Denial of Service

To mitigate resource exhaustion:

- Limit total concurrent reconstructions and aggregate buffered bytes
  - When sender identity is available, apply the same two limits per sender
- Enforce timeouts and size caps
- Validate segment counts (≤ 256)
- Consider rate-limiting at the transport layer (for example, via [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay) on Waku)

---

## Deployment Considerations

**Overhead:**

- Bandwidth overhead ≈ the predefined parity rate from parity (if enabled)
- Additional per-segment overhead ≤ **100 bytes** (protobuf + metadata)

**Network impact:**Ac

- Larger messages increase transport traffic and storage;
  operators **SHOULD** consider policy limits

**Compatibility:**

- Nodes that do **not** implement this specification cannot reconstruct any messages.

---

## References

1. [10/WAKU2 – Waku](https://rfc.vac.dev/waku/standards/core/10/waku2)
2. [11/WAKU2-RELAY – Relay](https://rfc.vac.dev/waku/standards/core/11/relay)
3. [14/WAKU2-MESSAGE – Message](https://rfc.vac.dev/waku/standards/core/14/message)
4. [64/WAKU2-NETWORK](https://rfc.vac.dev/waku/standards/core/64/network#message-size)
5. [nim-leopard](https://github.com/status-im/nim-leopard) – Nim bindings for Leopard-RS (Reed–Solomon)
6. [Leopard-RS](https://github.com/catid/leopard) – Fast Reed–Solomon erasure coding library
7. [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) – Key words for use in RFCs to Indicate Requirement Levels
