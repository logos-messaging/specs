---
title: Message Segmentation and Reconstruction over Waku
name: Message Segmentation and Reconstruction
tags: [waku-application, segmentation]
version: 0.1
status: draft
---

## Abstract

This specification defines an application-layer protocol for **segmentation** and **reconstruction** of messages carried over Waku when the original payload exceeds the maximum Waku's message size. Applications partition the payload into multiple Waku envelopes and reconstruct the original on receipt, even when segments arrive out of order or up to a **predefined percentage** of segments are lost. The protocol uses **Reed–Solomon** erasure coding for fault tolerance. Messages whose payload size is **≤ `segmentSize`** are sent unmodified.

## Motivation

Waku Relay deployments typically propagate envelopes up to **150 KB** as per [64/WAKU2-NETWORK - Message Size](https://rfc.vac.dev/waku/standards/core/64/network#message-size). To support larger application payloads, a segmentation layer is required. This specification enables larger messages by partitioning them into multiple envelopes and reconstructing them at the receiver. Erasure-coded parity segments provide resilience against partial loss or reordering.

## Terminology

- **original message**: the full application payload before segmentation.  
- **data segment**: one of the partitioned chunks of the original message payload.  
- **parity segment**: an erasure-coded segment derived from the set of data segments.  
- **segment message**: a Waku envelope whose `payload` field carries a serialized `SegmentMessageProto`.  
- **`segmentSize`**: configured maximum size in bytes of each data segment’s `payload` chunk (before protobuf serialization).  
- **sender public key**: the origin identifier used for indexing persistence.

The key words **“MUST”**, **“MUST NOT”**, **“REQUIRED”**, **“SHALL”**, **“SHALL NOT”**, **“SHOULD”**, **“SHOULD NOT”**, **“RECOMMENDED”**, **“NOT RECOMMENDED”**, **“MAY”**, and **“OPTIONAL”** in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Segmentation

### Sending

When the original payload exceeds `segmentSize`, the sender **MUST**:

- Compute a 32-byte `entire_message_hash = Keccak256(original_payload)`.
- Split the payload into one or more **data segments**, each of size up to `segmentSize` bytes.
- Optionally generate **parity segments** using Reed–Solomon erasure coding, at the predefined parity rate.  
  Implementations **MUST NOT** produce more than 256 total segments (data + parity).
- Encode each segment as a `SegmentMessageProto` with:
  - The `entire_message_hash`
  - Either data-segment indices (`segments_count`, `index`) or parity-segment indices (`parity_segments_count`, `parity_segment_index`)
  - The raw payload data
- Send all segments as individual Waku envelopes, preserving application-level metadata (e.g., content topic).

Messages smaller than or equal to `segmentSize` **SHALL** be transmitted unmodified.

### Receiving

Upon receiving a segmented message, the receiver **MUST**:

- Validate each segment according to [Wire Format → Validation](#validation).
- Persist received segments, indexed by `entire_message_hash` and sender.
- Attempt reconstruction when the number of available (data + parity) segments equals or exceeds the data segment count.
- Reconstruct by:
  - Concatenating data segments if all are present, or
  - Applying Reed–Solomon decoding if parity segments are available.
- Verify `Keccak256(reconstructed_payload)` matches `entire_message_hash`.  
  On mismatch, the message **MUST** be discarded and logged as invalid.
- Once verified, the reconstructed payload **SHALL** be delivered to the application.
- Incomplete reconstructions **SHOULD** be garbage-collected after a timeout.

## Wire Format

```protobuf
syntax = "proto3";

message SegmentMessageProto {
  // Keccak256(original payload), 32 bytes
  bytes  entire_message_hash    = 1;

  // Data segment indexing
  uint32 index                  = 2; // valid only if segments_count > 0
  uint32 segments_count         = 3; // number of data segments (>= 2)

  // Segment payload (data or parity shard)
  bytes  payload                = 4;

  // Parity segment indexing (used if segments_count == 0)
  uint32 parity_segment_index   = 5;
  uint32 parity_segments_count  = 6; // number of parity segments (> 0)
}
```

### Validation

Receivers **MUST** enforce:

- `entire_message_hash.length == 32`
- **Data segments:** `segments_count >= 2` **AND** `index < segments_count`
- **Parity segments:** `segments_count == 0` **AND** `parity_segments_count > 0` **AND** `parity_segment_index < parity_segments_count`

No other combinations are permitted.

---

## Implementation

### Reed–Solomon

Implementations that apply parity **SHALL** use fixed-size shards of length `segmentSize`.  
The last data chunk **MUST** be padded to `segmentSize` for encoding.  
The reference implementation uses **nim-leopard** (Leopard-RS) with a maximum of **256 total shards**.

### Storage / Persistence

Segments **MUST** be persisted (e.g., SQLite) and indexed by `entire_message_hash` and sender public key.  
Implementations **SHOULD** support:

- Duplicate detection and idempotent saves  
- Completion flags to prevent duplicate processing  
- Timeout-based cleanup of incomplete reconstructions  
- Per-sender quotas for stored bytes and concurrent reconstructions

### Configuration

**Required parameters:**

- `segmentSize` — **REQUIRED** configurable parameter; maximum size in bytes of each data segment's payload chunk (before protobuf serialization).

**Fixed parameters:**

- `parityRate` — fixed at **0.125** (12.5%)
- `maxTotalSegments` — **256** (library limitation for data + parity segments combined)

**Reconstruction capability:** With the predefined parity rate, reconstruction is possible if **all data segments** are received or if **any combination of data + parity** totals at least `dataSegments` (i.e., up to the predefined percentage of loss tolerated).

**API simplicity:** Libraries **SHOULD** require only `segmentSize` from the application for normal operation.

### Support

- **Language / Package:** Nim; **Nimble** package manager  
- **Intended for:** all Waku nodes at the application layer

---

## Security Considerations

### Privacy

`entire_message_hash` enables correlation of segments that belong to the same original message but does not reveal content.  
Applications **SHOULD** encrypt payloads before segmentation.  
Traffic analysis may still identify segmented flows.

### Integrity

Implementations **MUST** verify the Keccak256 hash post-reconstruction and discard on mismatch.  

### Denial of Service

To mitigate resource exhaustion:

- Limit concurrent reconstructions and per-sender storage  
- Enforce timeouts and size caps  
- Validate segment counts (≤ 256)  
- Consider rate-limiting using [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/waku/standards/core/17/rln-relay)

### Compatibility

Nodes that do **not** implement this specification cannot reconstruct large messages.  

---

## Deployment Considerations

**Overhead:**

- Bandwidth overhead ≈ the predefined parity rate from parity (if enabled)  
- Additional per-segment overhead ≤ **100 bytes** (protobuf + metadata)  

**Network impact:**

- Larger messages increase gossip traffic and storage; operators **SHOULD** consider policy limits

---

## References

1. [10/WAKU2 – Waku](https://rfc.vac.dev/waku/standards/core/10/waku2)  
2. [11/WAKU2-RELAY – Relay](https://rfc.vac.dev/waku/standards/core/11/relay)  
3. [14/WAKU2-MESSAGE – Message](https://rfc.vac.dev/waku/standards/core/14/message)
4. [64/WAKU2-NETWORK](https://rfc.vac.dev/waku/standards/core/64/network#message-size)
5. [nim-leopard](https://github.com/status-im/nim-leopard) – Nim bindings for Leopard-RS (Reed–Solomon)  
6. [Leopard-RS](https://github.com/catid/leopard) – Fast Reed–Solomon erasure coding library  
7. [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) – Key words for use in RFCs to Indicate Requirement Levels
