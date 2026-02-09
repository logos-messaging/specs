---
title: WakuChat
name: A private decentralized messaging protocol over Waku.
category: Standards Track
status: raw
tags: chat
editor: Jazz Alyxzander<jazz@status.im>
contributors:
---
# Abstract


# Background / Rationale / Motivation

Waku is a decentralized P2P protocol, which _______. 


## Definitions

This document makes use of the shared terminology defined in the [CHAT-DEFINITIONS](https://github.com/waku-org/specs/blob/jazzz/chatdefs/informational/chatdefs.md) specification.

The terms include:
- Invite
- OutOfBand
- Recipient
- Sender


# Theory / Semantics

This specification defines an implementation based on the Chat Protocol framework outlined [here](https://github.com/waku-org/specs/blob/jazzz/chat_framework/standards/application/chat-framework.md)

- **Discovery Protocol:** OOB Invite Links [TODO: Link to spec]()
- **Initialization Protocol:** [InboxV1](https://github.com/waku-org/specs/blob/jazzz/inbox_xk0/standards/application/inbox.md)
- **Conversation Protocols:** [PrivateV1](TODO)
- **Delivery Service:** Waku
- **Framing Strategy:** WapEnvelopes with protocol tagging


# Registration

A client is considered registered when it generates the required values:
- A static curve25519 keypair referred to as its `InstallationKey` or `IK`
-  

# Protocol Definitions

## Discovery Protocol
Clients are expected to find one another over existing established channels. 

Data Exchange is specified by using [Invite links](TODO).

Senders require the following data in order to proceed with initialization:
- Recipient IK
- Recipient EK
- delivery_address

Where EK is the ephemeral keys used in the initialization protocol.

## Initialization Protocol
This protocol uses `InboxV1` for handling of initial messages.


### Delivery Address
A clients inbox uses a static `delivery_address` so that Senders and Recipients can always derive it the same.

TODO: define delivery_address

### Conversation_id
A clients inbox uses a static `conversation_id` so that Senders and Recipients can always derive it the same.

It's defined as: `rhex(blake2b("inboxV1" || IK.public))`

TODO: define key format

## Conversation Protocols


# Transport Definitions

## Delivery Service

Waku is used as the Delivery service for all in band payloads. 

Clients MUST connect with with following parameters:
- ClusterId: 19
- ShardId: 0

Waku encryption is not used. Instead protocols rely on the application level encryption for confidentiality. 

### ContentTopics

The content topic of a payload is defined using [23/WAKU2-TOPICS](https://rfc.vac.dev/waku/informational/23/topics#content-topics) where:

- **application-name:** `wapchat`
- **version-of-the-application:** `0`
- **content-topic-name:** `chat-{rhex(blake2b(delivery_address, 8))}` #TODO: Decide on client load factor, and where to define prefix
- **encoding:** `proto`

This `content-topic-name` scheme is possible because all protocols used rely on inbox-based-addressing.

## Framing Strategy

All payloads are wrapped in a common envelope type and an encryption wrapper. This provides consistent parsing for clients across all protocols.


### Envelopes

Envelopes attach a reference to the state machine required to process the containing payload.

To process a payload, a client must know how a payload was encoded/encrypted and what encryption state to use to decrypt it.
As clients can support multiple sub-protocols simultaneously, its not always determinable how a message needs to be decrypted/handled. To resolve this a conversation_hint is included with every payload. 

### Conversation Hinting

Conversation_ids are sufficient to allow clients to determine how to decrypt a payload. However this would leak Conversation metadata, as observers could determine which payloads belonged together into a group. Instead a hint is used which allows clients to check if this payload is of interest without exposing the identifier.

ConversationHints are computed by using a salted hash of the `conversationId`. specifically defined as `rhex(blake2s(salt || conversation_id))`.

Clients can check if a conversation_hint is of interest to them, then retrieve the associated encryption state. Payloads with an unknown hint can be safely disregarded.

TODO: Can this be removed with ratcheting identifiers? Salting requires clients to check every conversation id.


## Wire Format Specification / Syntax

The wire format is specified using protocol buffers v3.

### Envelope
```
message WapEnvelopeV1 {
    
    // Indicates which conversation this message belongs to without revealing metadata 
    string conversation_hint = 1;
    // Used to compute the conversation_hint
    uint32 salt = 2;           
    // Data encoded according to the corresponding ConversationType
    bytes payload = 3;
}
```
This message is the root message of all payloads in the protocol.

- **conversation_hint:** This field contains a utf8 string representing the `conversation_hint`.
- **salt:** This field contains a 4byte salt used to compute the `conversation_hint`.
- **payload:** This field contains a protobuf encoded payload.


## Implementation Suggestions (optional)

### User level Conversations

Application developers should maintain standalone identifiers for user-level conversations that are separate from the protocol-level conversation_id. A single logical conversation from the user's perspective may utilize multiple ConversationTypes over time. As versions are considered different types, the underlying relationship is many to one. By maintaining application-level conversation identifiers, developers can provide users with consistent conversation continuity while the underlying protocol mechanisms handle version transitions and security upgrades transparently.


## Security/Privacy Considerations

Payloads inherit the privacy and security properties of the ConversationType used to send them. Please refer to the corresponding specifications when analyzing properties. 

### Inbox Privacy

Initial messages sent using this protocol do not benefit from recipient privacy. If an attacker learns of a clients inbox conversation_id, they can guess and check all messages for a matching conversation_hint. While not trivial, it must be assumed that attackers can determine if a client has receive a message.

In this case the Envelopes contain no sender information, so this does not leak social graph information.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

A list of references.
