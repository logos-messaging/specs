---
title: CHAT-DEFINITIONS
name: Shared definitions for Chat Protocols
category: Informational
tags: definitions, terminology, reference
editor: 
contributors:
---

## Abstract

This specification establishes a common vocabulary and standardized definitions for terms used across chat protocol specifications. It serves as a normative reference to ensure consistent interpretation of terminology across application level specifications.

## Background / Rationale / Motivation

Chat protocol specifications often employ similar terminology with subtle but important semantic differences. The absence of standardized definitions leads to ambiguity in specification interpretation and implementation inconsistencies across different protocol variants.

Establishing a shared definitions document provides several benefits:

1. **Semantic Consistency**: Ensures uniform interpretation of common terms across multiple specifications
2. **Implementation Clarity**: Reduces ambiguity for protocol implementers
3. **Specification Quality**: Enables more precise and concise specification language through reference to established definitions
4. **Ecosystem Coherence**: Promotes consistent understanding within the chat protocol ecosystem.

This specification provides normative definitions that other chat protocol specifications MAY reference to establish precise semantic meaning for common terminology.

## Theory / Semantics

### Definition Categories

Terms are organized into the following categories for clarity and ease of reference:

- **Roles**: Defined entity types that determine how a participant behaves within a communication protocol.
- **Message Types**: Classifications and categories of protocol messages
- **Transports**: Abstract methods of transmitting payloads

## Definitions

### Accounts + Identity

### Roles

**Sender**: A client which is pushing a payload on to the network, to one or more recipients.

**Recipient**: A client which is the intended receiver of a payload. In a group context there maybe multiple recipients 

**Participant**: A generic term for the rightful members of a conversation. Senders and Recipients are roles that participants can hold.

### Message Types

**Payload**: The encoded bytes as produced by a chat protocol. The term `message` is avoided due to conflicts with other layers.

**Frame**: A data structure which is required to implement a chat protocol. Frames are used to disambiguate the objects defined by chat protocols from transport 'messages' and application `content`. 

**Content**: The data that is intended for end-users or applications. Chat protocols transmit and route content between clients. 

**Content Type**: The structured format of content. These data structures represent specific encodings of Text, Image, Audio data. 

**Delivery Acknowledgement**: A notification from a receiving client to sender that their message was successfully received. While similar to a read-receipt delivery acknowledgements differ in that the acknowledgement originates based on the client, where read-receipts are fired when they are displayed to a user.

**Invite**: An protocol message from one client to another to establish a communication channel. Invites notify a client that someone wants to communicate with them, and provides the required information to do so.


### Transports

**Out-of-Band**: The transfer of information using a channel separate from the defined protocol channel. Data sent Out-of-Band is transmitted using an undefined mechanism. This is used when protocols requires information to be shared with another entity, but it does not describe how that should occur. The responsibility to define how this occurs is the implementer or other protocols in the suite. 


### Software Entities

**Client**: Software that implements a chat protocol. It is responsible for establishing and maintaining communication with the underlying network, performing protocol operations (e.g., encryption, key management, frame generation), and exposing an interface for higher-level software. A Client serves as the technical component that provides messaging capabilities to Applications.

**Application**: Software that integrates with a Client in order to send and receive content.



## Wire Format Specification / Syntax

This specification does not define wire format elements. All definitions are semantic and apply to the interpretation of terms used in other specifications that do define wire formats.

## Security/Privacy Considerations

This specification defines terminology only and does not introduce security or privacy considerations beyond those present in specifications that reference these definitions.

The definitions provided in this specification do not alter the security or privacy properties of implementations that adopt the defined terminology.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

[RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.