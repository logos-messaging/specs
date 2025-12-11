---
title: OP-CHAN
name: OpChan Decentralized Forum
status: raw
category: Standards Track
tags: waku
editor: 
contributors:
  - Jimmy Debe <jimmy@status.im>
---

## Abstract

This document specifies the architecture of OpChan.
OpChan is a decentralized forum application built on the Waku protocol.


## Background

OpChan supports ephemeral anonymous sessions and wallet-
backed identities, identity key delegation, and stores content locally
while distributing messages over the Waku network.
All user data persists locally
and syncs via Waku messages to peers.
- Authentication and user publishing are provided either by ed25519 stored in the client or
wallet signature.

## Terminology

- Forum: A discussion board (analogous to a forum or channel) that groups
  posts and moderation controls.
- Post: Top-level user content created within a forum.
- Comment: A reply to a `Post` or other `Comment` (threaded discussion).
- Participant: Any actor able to publish or consume messages (anonymous or
  wallet-backed).
- Anonymous session: A browser-generated ed25519 keypair used as
  identity for a user without a wallet.


## Specification

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”,
“SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and
“OPTIONAL” in this document are to be interpreted as described in [2119](https://www.ietf.org/rfc/rfc2119.txt).

OpChan uses [10/WAKU](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)
as the transport for peer-to-peer distribution of messages, 
as [14/WAKU-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md).
The messages are cryptographically signed using ed25519 keys generated locally in the broswer window.
Wallet users MAY create a token signed by the wallet's private key.

OpChan messages fall into two broad classes:

1. Content messages
2. Control messages

Message routing and
discovery are handled by [23/WAKU2-TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md).

### Message Format (high-level)

All messages transmitted over the Waku network MUST include the following envelope fields:

``` js

id: string 
type: enum { CELL_CREATED, POST_CREATED, COMMENT_CREATED, VOTE, BOOKMARK, MODERATION, DELEGATION }
timestamp: uint64
author: string // author public key or wallet address
signature: bytes // ed25519 signature of the canonical serialized body
delegationProof?: bytes // optional wallet signature authorizing the browser key
body: object //

```

Rules for signing:

1. Serialize `body` fields in stable key order (JSON canonical form or
   protobuf deterministic encoding).
2. Construct signing bytes with: `type`, `timestamp`, `author`
3. Sign with the browser key registered for the session.

OpChan MUST verify the `signature` against the `author` and,
when present, verify `delegationProof`.

### Identity

There are two types of identity options,
anonymous session and wallet delegation.
An anonymous session generates an ed25519 keypair locally with the client.
The key is used to sign all messages.
- The session may include a Call Sign. ***
A wallet delegation connects a user's wallet to OpChan.
The wallet produces a short-lived delegation signature as the key.
That delegation proof is attached to each message and
verified is by peers in the network.

#### Delegation flow:

1. Client generates a new ed25519 keypair and a delegation request.
2. Wallet signs the delegation request and returns a `delegationProof` with an
   expiration `timestamp`.
3. The client stores the `delegationProof`, `browserKey`,
and `expiry`.
The `delegationProof` SHOULD be included in outgoing messages until `expiry`.

- `delegationProof` verification,
verify the wallet signature over the delegation request matches the wallet public key,
the `author` public key in messages

### Moderation & Permissions


A post has metadata (id, name,
creator, admins, admins can moderate (hide, remove, or penalize content within the forum).

Moderation events are control messages with types like `MODERATION_ACTION`:

- action: enum { HIDE, REMOVE, PIN, UNPIN, CHANGE_OWNERSHIP },
 
Moderation messages MUST be signed by a recognized post admin.
Clients MUST validate the admin membership before applying moderation actions locally.


OpChan  

- posts
- comments
- votes
- users (identities, call signs, delegation proofs)
- bookmarks
- moderations

Incoming Waku messages are validated

If delegation is revoked, 
peers should ignore subsequent messages from the revoked delegation key.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References
- [10/WAKU](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)
- [14/WAKU-MESSAGES](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md)
- 


