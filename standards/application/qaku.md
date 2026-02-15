---
title: WAKU-APP-QAKU
name: Waku App Qaku
category: Standards Track
tags: waku
editor: 
contributors: 
- Jimmy Debe <jimmy@status.im>
---

## Abstract

This document describes the different components of the Qaku web application.
Qaku is a Waku web applications that utilizes Waku protocols to provide a private and
censorship-resistent question and answer service.

## Background

The Qaku web application Waku is a family of robust,
censorship-resistant communication protocols designed to enable privacy-focused messaging for the decentralized web.
Using Wak
Current Q&A platforms are controlled by centralized organizations who have the ability to remove, 
block and censor content published by any user.
This could cause a user to not express their ideas or ask reall questions with fear of prosecution by a thrid party. 

Qaku uses some [10/WAKU](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md) protocols to create a decentralized,
censorship-resistant Q&A forum.
The user can create a Q&A board with a identity address which publicly shared with other peers.
Questions and answers are sent directly to other peers without the need of storing and/or retrieving the content from a centralized provider.
Familiar features like upvoting, sharable links, and board moderation are also capable.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”,
“SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and
“OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

A Q&A board is a `content-topic` which is made available by on the Waku network.
Users subscribe to the `content-topic` using [12/WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md) and
to listen to new published messages.
Each message is identified by its `message_type`.
A collection of questions are the children of a `content-topic`,
while a collection of answers are the children of a question.
All clients MUST have an `identity` recognized by the network to publish new content.

### Identity

All clients MUST have `identity` consisting of a public address identifier in the form of a hexadecimal string.
Clients generate cryptographic key pair, public and private keys, and
the private key MAY be encrypt a password.
Public addresses are derived from the public keys as a hexadecimal string.
This address SHOULD be recongized by the other clients in the network to idenitify clients.

Each `identity` SHOULD be stored locally.
The address SHOULD be used to sign and verify messages.

### Creating a New Board

A client creating a new Q&A board is identitied as the `owner`.
The `owner` MAY share the `content-topic` with other clients, 
as the `owner`is the first client to know the `content-topic` for the new board.
At the time of creation, the `owner` SHOULD create a `ControlMessage` to configure a Q&A board 
and publish to the Waku network.


#### Control Message

This message contains the initial Q&A board configuration.
The `ControlMessage` SHOULD only be updated by the `owner`.

The `ControlMessage` is a JSON object:

-  It MUST include a title name
-  It MAY include a list of client public addresses identified as `admin`.
-  It SHOULD include the public address of the `owner`, identity explianed [below](#Identity).
-  It SHOULD include the start and end date for clients to publish a new post.

```js
{
  "ControlMessage" : [
    title: string,
    id: string,
    enabled: boolean,
    timestamp: number, // time the ControlMessage is created
    owner: string, // the public address of the owner
    admins: string[], 
    moderation: boolean, 
    description: string, 
    updated: number,
    startDate: number, // optional, enforced by the application
    allowsParticipantsReplies: number // optional, enforced by the application
  ]
}

```

`admins`:
- A list of client `identity` that are able to moderate the Q&A board

`id`:
- MAY be generated from the `title` value, `timestamp` value, and the `owner` public address value.
- RECOMMENDED hash using SHA256

`moderation`:

- If the question can be moderated by the `owner` and `admins`.
- It MUST be enforced by the application, see [messages](#Messsages)

 Clients subscribed to the Q&A board `content-topic` using the [12/WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md),
 MUST be able to view the `ControlMessage`.

### Other Message Types

The Waku network processes real-time message updates, which enables new questions and
answers to be acceisble quickly.
All questions and answers are published by the user or
the `owner` using [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md) on the Waku network.
Using the [13/WAKU-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md) protocol,
published messages are stored by Waku store nodes.
When clients subscribe to a Q&A board, the message history is retrieved from [13/WAKU-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md) nodes.

Clients messages published to a Q&A board's `content-topic` SHOULD remain accessible on the Waku network.
No client will be able to remove the message once published.
The `admins` and `owner` MAY moderate certain published messages making some content not visible to all users.
 
```js
{
  "ModerationMessage" : [
    "hash": string, // The hash of the QuestionMessage
    "moderated": boolean
  ]
}

```

#### Question Message

The `QuestionMessage` type is generated by clients publishing a new question to a Q&A board.
Each question MAY have an `author` whos value is the publishing client's `identity`.
The client does not need to be the `owner` of the Q&A board.
Also the `owner` can not retrict any `identity` from publishing a message.

```js
{
  "QuestionMessage" : [
    "question": string, // The question being published
    "timestamp": number, // the time when the question being published
    "author": identity // the identity of the client
  ]
}

```

#### Answered Message

The `AnswerMessage` type is generated by clients publishing a answer to a question. 
Each `AnswerMessage` SHOULD have a parent `QuestionMessage` type.


```js
{
  "AnsweredMessage" : [
    questionId: string, // The parent QuestionMessage
    hash: string, // Hash of the text
    text: string, // The answer content
    timestamp: number // The time of publishing the answer content
  ]  
}

```

#### Upvote Message

A `UpVoteMessage` is an optional feature that MUST be enforced by the web appliciation.

```js
{
  "UpVoteMessage" : [
    questionId: string, The hash of the question
    hash: string, // hash of the message
    type: string, // the upvote for which message type (AnswerMessage or QuestionMessage)
    timestamp: number, // the time of the upvote

  ]
}
```

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [10/WAKU](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)
- [12/WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md)
- [19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)
- [13/WAKU-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md)
