---
title: WEBRTC-API
name: Waku WebRTC API definition
category: Standards Track
status: raw
tags: [application, api, real time]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
- Oleksandr Kozlov <oleksandr@status.im>
- Hanno Cornelius <hanno@status.im>
---

## Table of contents

<!-- TOC -->
  * [Table of contents](#table-of-contents)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
  * [Syntax](#syntax)
  * [Waku WebRTC Signaling Protocol](#waku-webrtc-signaling-protocol)
    * [Overview](#overview)
    * [Terminology](#terminology)
    * [Content Topic Usage](#content-topic-usage)
    * [Wire Protocol](#wire-protocol)
      * [Message Fields](#message-fields)
    * [Signaling Flow](#signaling-flow)
      * [1. Offer (Connection Initiation)](#1-offer-connection-initiation)
      * [2. Answer (Connection Acceptance)](#2-answer-connection-acceptance)
      * [3. ICE Candidate (Trickle ICE Exchange)](#3-ice-candidate-trickle-ice-exchange)
      * [4. Reject (Connection Refusal)](#4-reject-connection-refusal)
      * [5. Busy (Busy Response)](#5-busy-busy-response)
      * [6. End (Session Termination)](#6-end-session-termination)
    * [Concurrent Sessions and Incoming Requests](#concurrent-sessions-and-incoming-requests)
    * [Security and Privacy Considerations](#security-and-privacy-considerations)
    * [Conclusion](#conclusion)
      * [Example Use Cases](#example-use-cases)
  * [Security/Privacy Considerations](#securityprivacy-considerations)
  * [Copyright](#copyright)
<!-- TOC -->

## Abstract

This document specifies the Waku WebRTC Signaling Protocol, which enables two peers in a Waku network to establish direct WebRTC peer-to-peer connections using Waku as a decentralized signaling channel. The protocol defines a standardized method for exchanging WebRTC handshake messages (SDP offers/answers and ICE candidates) over Waku, eliminating the need for centralized signaling servers.

This protocol is application-agnostic and can be used to establish WebRTC connections for any purpose including voice/video calls, real-time data exchange, and collaborative applications. It leverages Waku's decentralized, censorship-resistant properties for signaling while allowing high-bandwidth media and data to flow directly between peers once the connection is established.

## Motivation

WebRTC enables real-time peer-to-peer communication for voice, video, and data exchange, but traditionally requires centralized signaling servers to coordinate the initial handshake between peers. This creates several problems:

- **Centralization**: Reliance on signaling servers creates single points of failure and potential censorship
- **Privacy concerns**: Centralized servers can log connection attempts and metadata
- **Cost and complexity**: Maintaining signaling infrastructure adds operational overhead
- **Vendor lock-in**: Applications become dependent on specific signaling services

The Waku WebRTC Signaling Protocol addresses these issues by using Waku as a decentralized signaling channel. This approach provides:

- **Decentralization**: No single point of failure or control
- **Censorship resistance**: Leverages Waku's peer-to-peer network properties
- **Privacy preservation**: Signaling messages can be encrypted and routed through multiple nodes
- **Cost efficiency**: No need to maintain dedicated signaling infrastructure
- **Interoperability**: Standardized protocol enables cross-application compatibility

This protocol enables developers to build truly decentralized real-time applications while maintaining the performance benefits of direct peer-to-peer WebRTC connections.

## Syntax

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

# Waku WebRTC Signaling Protocol

## Overview

The Waku WebRTC Signaling Protocol defines a method for two peers in a Waku network to establish a direct WebRTC peer-to-peer connection. It leverages Waku as a decentralized signaling channel (in place of a traditional centralized WebRTC signaling server) to exchange the necessary handshake messages (SDP offers/answers and ICE candidates) for setting up a WebRTC connection. This protocol is application-agnostic – it can be used to set up connections for any purpose (e.g. audio calls, video calls, real-time data exchange, collaborative applications) without being tied to a specific calling UI or workflow.

In this protocol, Waku is used only for signaling (control messages). Once the WebRTC connection is established, high-bandwidth or real-time data (media streams, data channel messages, etc.) flow directly between the peers over the WebRTC peer connection. This approach offloads traffic from the Waku network and allows low-latency, peer-to-peer communication, while Waku provides censorship-resistant and privacy-preserving rendezvous for initiating the connection.

## Terminology

- **Peer / Participant**: A node or user participating in the protocol (each running a Waku client capable of WebRTC). In a given connection setup, one peer will act as the initiator (the one starting the WebRTC handshake) and the other as the responder (the one receiving a connection offer and answering it). These roles are per-session and not fixed; any peer may initiate a new connection to any other as long as both run the protocol.
- **Initiator**: The peer that begins the signaling handshake by sending an offer (equivalent to “calling party” in a call scenario).
- **Responder**: The peer that receives an offer and chooses whether to accept (answer) or refuse it (equivalent to “called party” in a call scenario).
- **Session**: A single WebRTC connection attempt (and, if accepted, the resulting WebRTC connection) between two peers. Each session is identified by a unique session ID generated by the initiator. All signaling messages related to that session carry this ID.
- **SDP**: Session Description Protocol, used in WebRTC to describe session capabilities (offers and answers).
- **ICE**: Interactive Connectivity Establishment, the mechanism for finding network paths between peers. Trickle ICE (as defined in RFC 8838) is used, meaning ICE candidates are exchanged incrementally as they are found, rather than all at once.
- **ICE Candidate**: A network address (IP:port, possibly through NAT) that one peer can use to try to connect to the other. Candidates are gathered and sent via signaling for connectivity checks.
- **WebRTC Data Channel**: A bidirectional data channel within the WebRTC peer connection for arbitrary data exchange. (Data channels and media streams are not sent over Waku; they are negotiated via this protocol and then sent directly peer-to-peer.)

## Content Topic Usage

Signaling messages in this protocol are carried as Waku messages on a designated content topic. To facilitate targeted delivery and privacy, each peer **SHOULD** listen on a content topic specific to itself, and initiators **SHOULD** send signaling messages to the content topic associated with the intended responder. For example, an application **MAY** derive a peer-specific topic by incorporating the peer’s ID or a derivative of it. One simple scheme is:

```text
/waku-webrtc/1/<peer-id>/proto
```

Where `<peer-id>` is a representation of the target peer’s Waku/libp2p peer ID (e.g. a base58 or base32 encoding). Using such a scheme, each peer subscribes (or registers a filter) to their own topic (containing their ID) and will receive only signaling messages directed to them. The initiator can publish an Offer to the responder’s topic, and the responder’s Answer (or Reject) can be published to the initiator’s topic.

**Privacy considerations:** Directly using a unique identifier (like a full peer ID) in the content topic can potentially allow observers to correlate messages with a specific user, since Waku relay or filter nodes will see the content topic of messages. To mitigate this, implementations **MAY** employ strategies such as hashing the peer ID or using bucketed topics (e.g. using a prefix of a hash of the peer ID) so that multiple peers share the same content topic, increasing anonymity. Another approach is to simply use a single, shared content topic for all WebRTC signaling (as was done in the Waku Phone prototype), embedding addressing info inside the message payload. A unified topic maximizes anonymity (all peers’ signals look alike) at the cost of efficiency (all clients receive all offers and must filter them). The choice of content topic strategy should balance privacy, scalability, and simplicity for the given application. In any case, encryption (discussed below) is critical so that only the intended recipient can read the signaling content, regardless of topic structure.

For the purposes of this specification, we assume a content topic prefix of:

```text
/waku-webrtc/1/signal/proto
```

and, if using peer-specific topics, appending or including an identifier for the target peer. All messages **MUST** be published with the Protocol Buffers (proto) encoding for the payload, as defined in the Wire Protocol section. Peers **SHOULD** mark these signaling messages as ephemeral (non-persisted) when sending, so that Waku nodes do not store them long-term (they are transient by nature, similar to volatile signaling in calls).

## Wire Protocol

Signaling messages are encoded in a single Protocol Buffers message type. All messages share this format, distinguished by a message type field. The protobuf schema (proto3) is defined below:

```proto
syntax = "proto3";

package waku.webrtc.v1;

message WakuWebRTCMessage {
  enum MessageType {
    OFFER = 0;       // Initiator to responder: proposes a new WebRTC connection
    ANSWER = 1;      // Responder to initiator: accepts and answers an OFFER
    CANDIDATE = 2;   // Either peer to the other: conveys an ICE candidate (Trickle ICE, RFC 8838)
    REJECT = 3;      // Responder to initiator: rejects an OFFER (connection refused)
    BUSY = 4;        // Responder to initiator: rejects an OFFER because the responder is busy
    END = 5;         // Either peer to the other: terminates an ongoing session (hang-up or cancel)
    // (Future extensions could add ACK or other message types if needed)
  }

  MessageType message_type = 1;
  bytes session_id = 2;               // Unique identifier for this session (generated by initiator)
  optional bytes initiator_peer_id = 10;  // Waku peer ID of the initiator (caller)
  optional bytes responder_peer_id = 11;  // Waku peer ID of the responder (callee)
  optional bytes webrtc_data = 20;    // Encoded WebRTC data (SDP offer/answer or ICE candidate).
}
```

### Message Fields

- **message_type**: Indicates which signaling message this is (Offer, Answer, Candidate, etc.) and therefore what role and data to expect.
- **session_id**: A random identifier for the session, generated by the initiator and included in all messages related to that session. It **MUST** be sufficiently unique (e.g. a UUID or 128-bit random value) such that concurrent sessions do not collide. Peers use this to tie incoming messages to the correct connection context.
- **initiator_peer_id** *(optional)*: The libp2p/Waku peer ID of the initiating peer. This can be included in the Offer message, and subsequent messages **MAY** include it for clarity. It is mainly for informational or validation purposes, since the initiator is implicitly the sender of the initial Offer.
- **responder_peer_id** *(optional)*: The peer ID of the responding peer (the target of the Offer). In an Offer, this identifies the intended recipient. In responses, it may be included as well. This is also primarily for information/routing confirmation, since the responder is typically the recipient of the initial Offer message.
- **webrtc_data** *(optional)*: Binary payload carrying the WebRTC signaling data when applicable. The content depends on `message_type`:
  - **For `OFFER`**: this field contains the WebRTC SDP Offer generated by the initiator (`RTCPeerConnection.createOffer()`), encoded as bytes. Typically this is the UTF-8 encoded SDP text (it **MAY** be compressed or binary encoded if both sides agree, but plain SDP text is simplest).
  - **For `ANSWER`**: this field contains the WebRTC SDP Answer generated by the responder (`RTCPeerConnection.createAnswer()`), encoded in the same manner as the offer.
  - **For `CANDIDATE`**: this field contains an ICE candidate to be added. The candidate can be encoded as the JSON or string representation of the ICE candidate (as returned by the WebRTC API). Each `CANDIDATE` message **SHOULD** carry a single ICE candidate. (Multiple `CANDIDATE` messages will be sent as new candidates are gathered.) Both peers may send `CANDIDATE` messages after sending or receiving an Offer/Answer, as they gather ICE candidates.
  - **For `REJECT`, `BUSY`, and `END`**: this field is unused and **SHOULD** be omitted (or empty). These message types carry no additional data beyond what is implied by their type.

All multi-byte fields (IDs, SDP, etc.) are transmitted as bytes. Text values like SDP can be UTF-8 encoded to bytes. Peers are responsible for serializing and deserializing the SDP and ICE data to/from this message format.

Because messages are sent via Waku, the `WakuWebRTCMessage` will itself be the payload of a Waku envelope (`WakuMessage`). That outer envelope’s `contentTopic` must be set appropriately (as discussed above), and the payload **SHOULD** be encrypted so that only the intended recipient can decode it. Waku supports both symmetric and asymmetric payload encryption. For example, if the initiator knows the responder’s static public key (from their peer ID), the Offer could be asymmetrically encrypted for that key so only the responder can decrypt it. Alternatively, the application can establish a symmetric key (out-of-band or via a separate handshake) and use that to encrypt all signaling messages for that session. At a minimum, messages **SHOULD** be encrypted using Waku Message Version 1 (which provides Whisper-style symmetric or asymmetric encryption). This ensures that even if an adversary observes the content topic or intercepts messages, they cannot read sensitive information like IP addresses in ICE candidates or session descriptions. (If no encryption key is provided, the signaling messages would be sent in plaintext, which is **NOT** recommended.)

## Signaling Flow

This section describes the typical flow of messages to establish and terminate a WebRTC connection between two peers. The protocol is symmetric in that either peer can play the initiator role at any time to start a new session. Implementations should be prepared to handle incoming connection offers while not interfering with any ongoing sessions (e.g., if a peer is already in a session, it might reject new incoming offers or signal busy).

Below, we outline each message type and its role in the flow:

### 1. Offer (Connection Initiation)

**Sent by:** The Initiator (to the Responder).

When a peer (the initiator) wants to establish a WebRTC connection to another peer, it begins by creating a new WebRTC offer and sending an `OFFER` message over Waku to the target peer:

- The initiator generates a fresh `session_id` (random unique bytes) to identify this connection attempt.
- The initiator obtains an SDP offer by creating a WebRTC peer connection object and calling `createOffer()`, then calling `setLocalDescription()` on that peer connection. The SDP offer (typically a large text blob containing media capabilities, supported codecs, etc.) is placed into the `webrtc_data` field of the `OFFER` message.
- The `message_type` is set to `OFFER`.
- The initiator **SHOULD** set its own peer ID in `initiator_peer_id`, and the target’s peer ID in `responder_peer_id`.
- The `OFFER` message is then published to the Waku content topic that the responder is expected to monitor (e.g., the topic derived from the responder’s peer ID, as discussed). It **SHOULD** be encrypted such that only the responder can decrypt it.

**Receiving an Offer:** The responder’s Waku node (or a mailserver, if using Store/Filter) will deliver the `OFFER` message to the responder if they are online (or when they come online, if using a store-and-forward and the message wasn’t marked ephemeral). Upon receiving an `OFFER`:

- The responder first verifies if they are indeed the intended recipient (e.g., by checking `responder_peer_id` if present matches its own ID). If not, or if the message was somehow erroneously received, it **MUST** be ignored.
- If the responder currently has no capacity or desire to accept a new connection (for example, the application is busy with another session or the user auto-rejects new connections), the responder **SHOULD** reply with a `REJECT` or `BUSY` message (see below) and **NOT** proceed with the WebRTC negotiation.
- If the responder is willing and able to consider the connection, it will process the SDP offer:
  - The responder creates a new `RTCPeerConnection` and calls `setRemoteDescription(offer)` with the received SDP offer data.
  - At this point, the WebRTC stack on the responder’s side may start gathering ICE candidates (for the upcoming answer) and may also be prepared to send an answer.
  - **Optional “ringing/progress”:** In user-facing call applications, at this stage the responder might alert the user (e.g., ringing). The protocol does not require sending any immediate response prior to the final answer; however, an application **MAY** choose to inform the initiator that the offer was received and is being processed. In a telephony context, this was represented by a `RINGING` message in Waku Phone (which let the caller know the callee’s device is alerting). In the generic protocol, there is no strict analogue, but an implementation could define a custom provisional response if needed. In most cases, the absence of an immediate `REJECT` and the eventual `ANSWER` serves as implicit feedback (or the initiator can implement a timeout). For simplicity, this specification does not include a formal `RINGING` message type.

The responder typically will now wait for user confirmation or some condition to decide whether to accept or reject the connection. This leads to one of two outcomes: an Answer or a Reject/Busy.

### 2. Answer (Connection Acceptance)

**Sent by:** The Responder (to the Initiator), if the responder accepts the connection.

When the target peer decides to accept the incoming connection (e.g., the user answered the call, or the application auto-accepts), the responder generates an SDP answer and sends an `ANSWER` message:

- The responder (having already set the initiator’s offer as the remote description on its `RTCPeerConnection`) now calls `createAnswer()`, then `setLocalDescription()` on its connection. This produces an SDP answer describing the responder’s capabilities that are compatible with the offer.
- The responder prepares an `ANSWER` message with:
  - `message_type = ANSWER`.
  - The same `session_id` as the corresponding Offer (copied exactly).
  - It may include the `initiator_peer_id` and `responder_peer_id` (swapped relative to the Offer’s perspective, but including them is optional as the `session_id` identifies the context).
  - The `webrtc_data` field containing the SDP answer (UTF-8 bytes).
- The `ANSWER` message is sent to the initiator’s content topic (i.e., the initiator’s Waku address). This notifies the initiator that the responder agreed and is ready to complete the connection. This message **SHOULD** be encrypted so that only the initiator can read it.

**Receiving an Answer:** The initiator, upon receiving the `ANSWER` message:

- Verifies the message is for the correct session (matching `session_id` of an outstanding Offer it sent) and from the expected responder (if IDs are present).
- Takes the SDP answer from `webrtc_data` and calls `setRemoteDescription(answer)` on its `RTCPeerConnection`. At this point, the WebRTC peer connection on the initiator’s side transitions to a connected or connecting state. Both peers now have each other’s SDP and can proceed with ICE negotiation to establish the actual transport channels.
- The initiator does not send a specific acknowledgment for the answer via Waku; the WebRTC connection process itself (ICE connectivity establishment) will indicate success or failure. Once the answer is applied, both sides will continue to exchange ICE candidates (if using trickle ICE) and eventually, if network conditions allow, the peer connection will go into a “connected” state (ICE Completed/Nominated, DTLS handshake done).

At this stage, the WebRTC connection is considered established from a signaling perspective. Both peers should have their `RTCPeerConnection` objects in place with local and remote descriptions set. The next phase is exchanging ICE candidates (if not all were already included in SDP or exchanged) until the transport is up. The application can now also open additional data channels or send media.

### 3. ICE Candidate (Trickle ICE Exchange)

**Sent by:** Either peer (both Initiator and Responder), as needed.

To expedite connection establishment, this protocol supports Trickle ICE (incremental candidate delivery). Rather than waiting to gather all ICE candidates before exchanging, peers send candidates to each other as they are found. The `CANDIDATE` message type is used for this purpose and can be sent multiple times by each side.

- As soon as a peer (initiator or responder) has set a local description (offer or answer), its WebRTC ICE agent begins gathering network candidates (e.g., host, STUN reflexive, TURN relay addresses). For each candidate gathered, the peer **SHOULD** send a Waku `CANDIDATE` message to its counterpart:
  - `message_type = CANDIDATE`.
  - `session_id` set to the session’s ID (so the other side knows which ongoing handshake this relates to).
  - `webrtc_data` containing the candidate information. This can be the result of calling `JSON.stringify(candidate)` on the `RTCIceCandidate` object, or constructing a small JSON with fields like `candidate` (the ICE candidate string), `sdpMid` and `sdpMLineIndex` as needed to allow the remote side to add it. (The exact format is left to the implementation; it just needs to convey all the necessary ICE candidate fields.)
  - Typically, `initiator_peer_id` and `responder_peer_id` are not needed in `CANDIDATE` messages (the `session_id` is sufficient to demux), and they can be left unset to reduce size. If the message is being sent on a peer-specific topic, the recipient is unambiguous.
- The peer sends the `CANDIDATE` message via Waku to the other party’s topic. These messages should also be encrypted. Multiple candidate messages will likely be sent by each side, asynchronously, as ICE continues to gather.

**Receiving a Candidate:** When a peer receives a `CANDIDATE` message:

- It checks the `session_id` and finds the corresponding `RTCPeerConnection` for that session (one that has been initialized with an offer/answer already).
- It then takes the candidate from the `webrtc_data` and adds it to its WebRTC connection via `addIceCandidate()` API.
- This allows the ICE agent to consider that remote candidate in attempting to find a path. If the candidate is valid and connectivity can be achieved, eventually the ICE state will turn to “connected” or “completed” on both sides, finalizing the peer-to-peer channel setup.

This trickle process continues until either:

- The ICE agents find a viable pair of candidates and establish a connection (at which point additional candidates might be ignored or can be optionally still added for backup paths).
- Or it times out / fails to find any connection (in which case the application might choose to end the session).

**Note:** Because Waku is an asynchronous medium and not guaranteed to deliver messages in order, it’s possible that a `CANDIDATE` arrives before the corresponding Offer/Answer that provides the initial SDP. Implementations should be prepared to handle out-of-order events. A candidate received before the peer’s `RTCPeerConnection` is ready (i.e., before an Offer/Answer has been set) can be queued or temporarily discarded until the SDP is in place.

### 4. Reject (Connection Refusal)

**Sent by:** The Responder (to the Initiator), if the responder does not accept the offer.

If the responder decides to decline the incoming connection request (e.g., the user rejected the call, or the application policy disallows it), the responder sends a `REJECT` message as a final response:

- `message_type = REJECT`.
- It should include the same `session_id` from the Offer it is rejecting.
- `initiator_peer_id` and `responder_peer_id` can be included (to identify the parties), but are not strictly required.
- No `webrtc_data` is included (the field is empty/omitted).
- The `REJECT` is sent to the initiator’s Waku topic (so that the initiator will receive it).

**Receiving a Reject:** The initiator, upon receiving a `REJECT` for a session it started:

- Stops waiting for an answer on that session (if it was still waiting).
- Cleans up any state for that attempted session (e.g., can close the `RTCPeerConnection` it had created for the offer, since no connection will be established).
- Notifies the application/user that the attempt was refused.
- The initiator **MUST NOT** continue to send candidates or any further messages for that session once a `REJECT` is received. The session is considered terminated at this point.

### 5. Busy (Busy Response)

**Sent by:** The Responder (to the Initiator), as a specialized refusal indicating the responder is currently busy.

`BUSY` is a specific kind of rejection used when the responder is unable to accept the offer because it’s occupied, for example, already in another session. The `BUSY` message is very similar to `REJECT` in its structure and handling, but it provides a hint about the reason:

- `message_type = BUSY`.
- It uses the same `session_id` of the incoming offer.
- No `webrtc_data` (and peer ID fields optional as usual).
- Sent to the initiator’s topic.

**Receiving a Busy:** The initiator handles a `BUSY` message essentially the same way as a `REJECT`:

- Terminate the pending session attempt and free associated resources.
- Inform the application that the peer is busy.
- The initiator **SHOULD NOT** immediately retry the connection, as the peer explicitly signaled it’s busy. It may wait or let the user decide to call again later.

Including a `BUSY` distinct from `REJECT` is optional in implementations. It is provided to allow a nicer user experience.

### 6. End (Session Termination)

**Sent by:** Either peer (to the other), at any time after a session has been initiated, to abort or terminate the session.

The `END` message is used to gracefully terminate an ongoing or pending WebRTC session. Either side can send an `END`:

- An initiator may send `END` if they decide to cancel the request before it’s answered (e.g., user hung up before the call was answered, or a timeout on waiting for answer was reached). In this case, `END` functions as a “cancel” signal.
- Either peer (initiator or responder) may send `END` during an active WebRTC connection (after answer, when the peers are in call or exchanging data) to hang up or disconnect the session.
- A responder might send `END` if it had sent an answer but then wants to terminate.

**To send an END:**

- `message_type = END`.
- Include the relevant `session_id`.
- No `webrtc_data` needed.
- The message is sent to the other party’s Waku topic.

**Receiving an End:** When a peer receives an `END` message for a session:

- It should immediately consider that session terminated. If a WebRTC peer connection was established, the application should close it (e.g., call `RTCPeerConnection.close()` and stop any media streams). If the handshake was still in progress, it means the attempt was canceled.
- The peer should release any related resources.
- If the session was active, the user or application should be informed that the peer has disconnected.

In summary, `END` is a catch-all termination signal that can be used in any state of the session to signal a halt. After sending or receiving `END`, both sides should clean up. No further signaling messages for that session should occur. (If a peer still wants to reconnect or try again, it would start a brand new session with a new `session_id` and Offer.)

## Concurrent Sessions and Incoming Requests

The protocol allows multiple independent sessions to be negotiated concurrently (either with different peers or even multiple with the same peer, if the application supports it). Each session is isolated by its unique `session_id`. An implementation can handle these in parallel as needed. However, an application may choose to restrict usage (for example, a phone-call app likely only allows one active call at a time). In such cases, if a new Offer comes in while the peer is busy, the peer should send `BUSY` or automatically reject it as described.

It’s important that an implementation provides an event-driven API to handle incoming offers. For example, a library implementing this spec might emit an event like `incomingOffer` with details (`session_id`, initiator’s ID, maybe proposed media parameters) when an `OFFER` message is received, so that the application can decide to accept or reject it. Likewise, the API should allow the application to know when a session is established or ended (e.g., events for “connection established” and “connection closed”).

## Security and Privacy Considerations

Establishing a WebRTC connection involves exchanging sensitive information such as network addresses (ICE candidates can reveal IP addresses) and capabilities. Using Waku as the signaling transport provides a decentralized channel, but security measures must be taken:

- **Encryption:** All signaling messages **SHOULD** be encrypted so that only the intended peer can read them. Waku payload encryption (Message Version 1) supports both asymmetric (per-recipient public key) and symmetric encryption. For example, if peer A knows peer B’s Waku public key (identity key), A can encrypt the `OFFER` for B using asymmetric encryption (so only B can decrypt). B’s response can be encrypted to A’s public key. If exchanging public keys is not feasible, a shared symmetric key can be agreed upon (through a pre-shared secret or a Diffie-Hellman exchange via some other secure channel) and then used to encrypt/decrypt all messages for that session (or even for all sessions between A and B). Without encryption, an attacker observing Waku network traffic could glean who is trying to connect to whom (from content topics or meta-data) and read the SDP which could leak IP addresses or fingerprints. Thus, encryption is a **MUST** for privacy in most use cases.
- **Authentication:** The protocol as specified does not include an explicit authentication or verification step of the peers’ identity beyond knowing the peer ID/public key. If a Waku message is encrypted to a peer’s key, that provides assurance it came from someone with the corresponding private key (since only they could craft a message you can decrypt). However, peers should still be mindful of potential impersonation. Applications that require stronger identity verification may layer additional authentication (e.g., a challenge/response or signing the SDP).
- **Denial-of-Service (DoS):** Because this protocol is open (anyone can send an `OFFER` to a peer’s topic if they know the peer ID), there is potential for spam or DoS by flooding offers. Implementers should also consider local policies: e.g., rate-limit incoming offers, require some form of introduction or pre-shared contact to accept offers, etc. An implementation might mitigate this by screening offers (for example, only allow incoming from known peers or use a reputation system).
- **Privacy of Content Topics:** If using peer-specific content topics (as recommended for efficiency), note that the content topic itself reveals a link to a peer’s ID. A third-party node can see that someone is sending messages to `/waku-webrtc/1/<peer-id>/proto`. This could be used to infer that `<peer-id>` is active or being contacted. It can also link the initiator and responder if the timing of messages on their respective topics is observed. To reduce this metadata leakage, implementations can use the techniques mentioned (topic hashing/bucketing or a shared topic for all signals). Additionally, Waku’s Relay protocol (gossip) provides some recipient anonymity by not revealing to intermediate nodes exactly which peer is the recipient, whereas Filter/Store protocols require telling a server the topics of interest (hence revealing the peer’s content topic subscription). Using a generic content topic for all WebRTC signaling (and filtering at the client level by peer IDs inside the message) maximizes anonymity but at a performance cost.
- **Ephemeral Signaling:** Signaling messages typically have short usefulness (only during call setup). It’s recommended to mark them as ephemeral (non-persistent) so that Waku nodes do not store them beyond a short time. This reduces the risk of someone later retrieving historical call attempts via the Store protocol. It also means if a peer was offline during a call attempt, they might never see the offer (which is acceptable, akin to a “missed call”). If an application did want a “missed connection” notification, it could deliberately not mark the message ephemeral; but this should be weighed against privacy. In most cases, real-time connections should be ephemeral.
- **WebRTC Security:** Once the peer connection is established, the media and data channels are secured by DTLS/SRTP in WebRTC. That encryption is end-to-end between the peers. This protocol does not alter or degrade the security of the WebRTC channel itself. However, any data sent over the peer connection is out of scope for Waku’s protections; if additional privacy or E2E encryption at the application level is needed (for example, E2E encrypting the media), the application should implement that on top of WebRTC as usual.

## Conclusion

The Waku WebRTC Signaling Protocol provides a way for any two Waku peers to negotiate a direct WebRTC connection without centralized servers, making it possible to build decentralized real-time applications (voice/video calls, peer-to-peer chats, collaborative tools, etc.). By abstracting the signaling into a generic protocol, it enables reuse across many application domains. A high-level API might look like `waku.dialWebRTC(peerId, options)` for initiating a connection and event handlers for incoming offers, abstracting the details of message passing.

With this protocol in place, developers can focus on the application logic (what to do with the peer connection once established) rather than the complexities of how to get two browsers or nodes connected. Whether it’s establishing a video call or syncing a document in real-time, the same Waku signaling handshake can be used under the hood. All of this is accomplished in a decentralized manner, benefiting from Waku’s resilience and privacy features.

### Example Use Cases

- A decentralized video calling dApp can use this protocol to ring another user and set up a WebRTC stream, without any server for call signaling.
- A collaborative editing app can use it to connect two peers and then open a data channel for live text synchronization.
- A file transfer between two users can be negotiated over Waku, then the bulk data sent peer-to-peer via a data channel.

> **Note:** This is a flexible protocol. Implementers should test various network conditions (NAT types, need for TURN, etc.) when using it. Waku does not inherently provide STUN/TURN functionality, so in situations of heavy NAT, a TURN server might still be required for the WebRTC media path. The signaling will work regardless, but connectivity might need that fallback. Including STUN/TURN servers in the peer connection configuration (as part of the SDP) is therefore recommended for real-world usage, or future extensions of Waku might offer decentralized ICE relay services.

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
