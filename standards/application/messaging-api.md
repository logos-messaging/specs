---
title: MESSAGING-API
name: Messaging API definition
category: Standards Track
tags: [reliability, application, api, protocol composition]
editor: Oleksandr Kozlov <oleksandr@status.im>
contributors:
- Oleksandr Kozlov <oleksandr@status.im>
- Prem Chaitanya Prathi <prem@status.im>
---

## Table of Contents

- [Abstract](#abstract)
- [Design Requirements](#design-requirements)
- [API design](#api-design)
  - [Requirements](#requirements)
  - [Initial configuration](#initial-configuration)
  - [Send](#send)
  - [Subscribe](#subscribe)
  - [Message Storage](#message-storage)
  - [Health Indicator](#health-indicator)
  - [Event Source](#event-source)

## Abstract

This document specifies an Application Programming Interface (API) that is RECOMMENDED for developers of the [WAKU2](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/10/waku2.md) clients to implement,
and for consumers to use as a single entry point to its functionalities.

This API defines the RECOMMENDED interface for leveraging Waku protocols to send and receive messages. 
Applications SHOULD use it to access capabilities for peer discovery, message routing, and reliability.

This document defines the specifications and guidelines necessary for implementing the API,
ensuring interoperability and consistency across the Waku protocol family.

## Design Requirements
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

### Definitions

This section defines key terms and concepts used throughout this document.
Each term is detailed in dedicated subsections to ensure clarity and consistency in interpretation.

`Multiaddr` - a self-describing format for encoding network address of a remote peer as per [libp2p addressing](https://github.com/libp2p/specs/blob/6d38f88f7b2d16b0e4489298bcd0737a6d704f7e/addressing/README.md) specification.

### Motivation

Real-world application development has exposed challenges in achieving comprehensive usability guarantees for core Waku protocols.
Although [P2P-RELIABILITY](./p2p-reliability.md) have been introduced to enhance protocol robustness,
the fragmented landscape of guidelines and the requirement to re-implement reliability strategies across diverse use cases remain evident.

A significant challenge has been the lack of a unified abstraction for message exchange.
Developers require a straightforward mechanism to send, receive,
and track messages—whether originating from a node or propagated throughout its connected network.

## API design

### Requirements

This API is designed for generic use and ease across all implementations and traditional `REST` architectures.
From this point forward,
relevant code blocks will be provided in `JSON` format, `HTTP` format or by using `TypeScript` syntax.

### Initial configuration

The `Messaging API` is an abstraction layer built upon the basic functionality provided by a node.  
The configuration definitions that follow omit implementation-specific details,
focusing exclusively on settings explicitly required by the `Messaging API`.

```typescript
{
  mode: "edge" | "relay";
  clusterId: number;
  shards: number[];
  shardsUnderCluster?: number;
  staticStoreNode?: string[];
  preferredServiceNodes?: string[];
  bootstrapNodes: string[];
  storeConfirmation?: boolean;
  receiveConfirmation?: boolean;
  confirmContentTopics?: string[];
}
```

#### Properties

##### `mode`
This property defines behavior of the node and MUST be specified.

If the node is operating in `edge` mode, it MUST:
- Employ [LIGHTPUSH](../standards/core/lightpush.md) for sending messages.
- Utilize [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) for receiving messages.
- Employ [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/f08de108457eed828dadbd36339433c586701267/waku/standards/core/34/peer-exchange.md#abstract) to discover peers.
- Utilize [STORE](../standards/core/store.md) to obtain message acknowledgements, as described in the Message Storage API section.

If the node is configured in `relay` mode, it MUST:
- Deploy the [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) protocol.
- Host endpoints for [LIGHTPUSH](../standards/core/lightpush.md) and [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md).
- Serve the [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/f08de108457eed828dadbd36339433c586701267/waku/standards/core/34/peer-exchange.md#abstract) protocol.

`edge` mode SHOULD be used if node functions in resource restricted environment,
where as `relay` SHOULD be used if node has no hard restrictions.

##### `clusterId`
This property MUST be provided.
It signifies which cluster a node MUST be operating at as per [RELAY-SHARDING](https://github.com/waku-org/specs/blob/186ce335667bdfdb6b2ce69ad7b2a3a3791b1ba6/standards/core/relay-sharding.md).

##### `shards`
This property MUST be provided.
An array of shard under a specified cluster that node MUST operate at as per [RELAY-SHARDING](https://github.com/waku-org/specs/blob/186ce335667bdfdb6b2ce69ad7b2a3a3791b1ba6/standards/core/relay-sharding.md).

##### `shardsUnderCluster`
This is an optional property that MUST be a positive integer.
If not provided, it defaults to 8.
This property is used to automatically map a provided `contentTopic` to the appropriate `shards` under the specified `clusterId`.
For further details, refer to the [RELAY-SHARDING](https://github.com/waku-org/specs/blob/master/standards/core/relay-sharding.md#content-topics-format-for-autosharding) specification.

##### `staticStoreNode`
A list of `Multiaddr` addresses to remote peers that MUST be used for retrieving past messages or performing infrequent queries using the [STORE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/13/store.md) protocol.
If not provided, the implementation SHOULD utilize either `preferredServiceNodes` or nodes discovered via the network.

##### `preferredServiceNodes`
A list of `Multiaddr` addresses to remote peers that SHOULD be used for obtaining various network resources.

These resources MAY include:
- infrequent [STORE](../standards/core/store.md) queries;
- a [LIGHTPUSH](../standards/core/lightpush.md) endpoint for broadcasting data;
- a [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) endpoint for receiving circulating data.

If preferred nodes are not provided or are unreachable (i.e., a connection cannot be established due to errors or timeouts),
the implementation SHOULD use nodes discovered through the network.

##### `bootstrapNodes`
This property MUST be provided.

A list of `Multiaddr` addresses to remote peers that MUST be used for any applicable method of discovery that MAY include:
- [DNS Discovery](https://eips.ethereum.org/EIPS/eip-1459);
- [DISCV5](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/33/discv5.md);
- [PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/34/peer-exchange.md).

##### `storeConfirmation`
This optional property defaults to `true`.
- If set to `true`, the `stored` field on `MessageRecord` MUST be populated, and implementations MUST employ store‑based reliability as specified in [P2P‑RELIABILITY](./p2p‑reliability.md).
- If set to `false`, the `stored` field on `MessageRecord` MUST NOT be populated.
For more information about the [STORE](../standards/core/store.md) query and the `MessageRecord`, refer to the `Message Storage API` section.

##### `receiveConfirmation`
This optional property defaults to `true`.
- If set to `true`, the node MUST initiate message‑reception detection as specified in [P2P‑RELIABILITY](./p2p‑reliability.md) and populate the `received` field on `MessageRecord`.  
- If set to `false`, the `received` field MUST NOT be populated, unless the `Subscribe API` is invoked.

##### `confirmContentTopics`
An optional property that SHOULD be provided in conjunction with either `storeConfirmation` or `receiveConfirmation`.
It includes an array of `contentTopics` that the node MUST start monitoring in the background upon initialization,
as defined in the [TOPICS](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/informational/23/topics.md#content-topic-format) specification.

#### Programmatic API / REST API

No methods or `REST` endpoints are provided for initial configuration.
Instead, the initial configuration MUST be supplied at node creation time according to the guidelines of the chosen implementation.

### Send

The `Send API` is responsible for broadcasting messages across the network using the configured protocol.
The node SHOULD select the appropriate protocol based on its configuration and the protocols that are mounted:
- If the initial configuration specifies mode: `edge`, then [LIGHTPUSH](../standards/core/lightpush.md) MUST be used to send messages.
- If the initial configuration specifies mode: `relay`, then [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) MUST be used to send messages.

The `Send API` is also responsible for ensuring message delivery and MUST attempt retries:
- Whenever message loss is detected, in accordance with [P2P‑RELIABILITY](./p2p‑reliability.md).  
- Implementations MAY choose their own backoff timing, maximum number of retry attempts and prioritization of requests.

#### Programmatic API

```typescript
type RequestID = string;

type Message = {
  meta?: UInt8Array;
  timestamp?: number;
  ephemeral?: boolean;
  payload: UInt8Array;
};

interface ISend {
  public send(encoder: Encoder, message: Message): RequestID;
  public cancelRequest(requestId: RequestID): void;
  public getPendingRequests(): RequestID[];
};
```

##### `Encoder`
The `Encoder` object encapsulates the logic for message handling.
It MUST ensure that messages are encoded into the appropriate format for transmission.
Additionally, it MUST provide information regarding the `pubsubTopic` and `contentTopic` to which messages SHOULD be sent.
The specific types of encoders, their extended functionalities, and the mechanisms for their instantiation are considered out of scope for this document.

##### `Message`
A `Message` object MUST conform to the [MESSAGE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/14/message.md) specification.
- The payload property MUST be provided.
- All other properties are optional; if omitted, they SHOULD be supplied by the `Encoder`.

##### `RequestID`
A `RequestID` is a `GUID` string that uniquely identifies a message send operation.
This `GUID` MAY be used to cancel a scheduled request or to retrieve information about the associated message from the `Message Storage API`.

##### `send`
An `Encoder` and a `Message` MUST be provided as parameters.
When invoked, the method schedules the message to be sent via [LIGHTPUSH](../standards/core/lightpush.md) or [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) based on the node’s configuration.
The method MUST throw an error if the `pubsubTopic` provided by the `Encoder` is not supported by the underlying node.
The method MUST return the associated `RequestID` for the scheduled send attempt.
If the send operation fails, it MAY be retried from the `Message Storage API` using the `RequestID`.
Additionally, send operation can be monitored by subscribing to `message:sent` or `message:error` events described in `Event Source API` below.

##### `cancelRequest`
Cancels a scheduled send attempt associated with a given `RequestID`.
A call to this method MUST be ignored if the request has already been fulfilled (i.e., if it has succeeded, failed, or been canceled previously) or if the provided `RequestID` is invalid.

##### `getPendingRequests`
Returns an array of unique `RequestID` values corresponding to send operations that are still pending.
A request is considered pending if it has not yet succeeded,
and it may be actively retrying or scheduled for sending but has not yet been served.
No duplicate `RequestID` values MUST appear in the returned array.

#### REST API

##### `send`
Request:
```http
POST /send HTTP/1.1
Accept: application/json
Content-Type: application/json

{
  "timestamp": 0,
  "ephemeral": false,
  "meta": "ZXhhbXBsZQ==",
  "pubsubTopic": "string",
  "contentTopic": "string",
  "payload": "ZXhhbXBsZQ=="
}
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "requestId": "9978c4f1-4465-4537-9d95-0cf016fb1080"
}
```

Example for `404`:
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "Failed to send message. Target pubsubTopic 'string' not supported."
}
```

##### `cancelRequest`
Request:
```http
POST /send/cancel HTTP/1.1
Accept: application/json
Content-Type: application/json

{
  "requestId": "9978c4f1-4465-4537-9d95-0cf016fb1080"
}
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "ok"
}
```

##### `getPendingRequests`
Request:
```http
GET /send/requests HTTP/1.1
Accept: application/json
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  "9978c4f1-4465-4537-9d95-0cf016fb1080"
]
```

### Subscribe

The `Subscribe API` SHOULD be used to initiate a subscription over [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) or [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) for continuous message retrieval.

Upon invocation of the Subscribe API, implementations MUST:
- Establish a subscription via either the [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) protocol or the [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) protocol, based on the node’s configured reception method.  
- Ensure active subscriptions comply with the recommendations defined in [P2P‑RELIABILITY](./p2p‑reliability.md).

Once the `Subscribe API` is called, the `received` property in the `MessageRecord` (as described in the `Message Storage API`) MUST be populated.

When the subscription is active, the API MUST trigger the `message:*` event group as described in the `Event Source API` section.

#### Programmatic API

```typescript
type SubscriptionID = string;

type SubscriptionListener = (message: MessageRecord) => void | Promise<void>;

interface ISubscribe {
  public subscribe(decoder: Decoder, fn: SubscriptionListener): SubscriptionID;
  public unsubscribe(id: SubscriptionID): void;
  public getActiveSubscriptions(): SubscriptionID[];
}
```

##### `SubscriptionID`
A unique identifier (GUID string) representing a subscription.
This ID MUST be used to manage active subscriptions,
allowing them to be terminated when they are no longer needed.

##### `SubscriptionListener`
A callback function that is invoked whenever a new `MessageRecord` is received.
It processes the incoming message and MAY return a `Promise` to support asynchronous handling.
The `MessageRecord` is defined in the `Message Storage API` section.

##### `subscribe`
Registers a new subscription to listen for messages processed by a given `Decoder`.
The subscription is based on the `pubsubTopic` and `contentTopic` provided by the `Decoder`,
which determine the scope of the subscription and the messages to be received.
The subscription is uniquely identified by a `SubscriptionID`,
which MUST be used for managing the subscription (e.g., for later unsubscribing).
The method MUST throw an error if the `pubsubTopic` provided by the `Decoder` is not supported by the underlying node.
No specific order is guaranteed for callback invocation;
subscription callbacks MUST be executed as messages are received from the protocol in use.

##### `unsubscribe`
Removes a previously created subscription identified by its unique `SubscriptionID`.
Once unsubscribed, the associated `SubscriptionListener` MUST NOT receive any further messages.
If the provided `SubscriptionID` does not correspond to an active subscription,
the call MUST be ignored.
If recurring [STORE](../standards/core/store.md) query was initiated by this `SubscriptionID`, it MUST be stopped.

##### `getActiveSubscriptions`
Returns an array of unique `SubscriptionID` values representing all currently active subscriptions.
Each identifier corresponds to a subscription that is in effect,
and no duplicate values MUST be present in the returned array.

##### `Decoder`
The `Decoder` object encapsulates the logic for processing received messages.
It MUST ensure that incoming messages are decoded from their received format into a usable structure.
Additionally, it MUST filter messages based on the `contentTopic` to ensure that only relevant messages are processed.
The specific types of decoders, their extended functionalities,
and their instantiation mechanisms are considered out of scope for this document.

#### REST API

Reading messages is performed via the `/messages?subscriptionId="<GUID>"` endpoint,
as detailed in the `Message Storage API` section.

##### `subscribe`
Request:
```http
POST /subscribe HTTP/1.1
Accept: application/json
Content-Type: application/json

{
  "pubsubTopic": "/waku/2/rs/0/2",
  "contentTopic": "/example/2/rfc/proto"
}
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "subscriptionId": "9978c4f1-4465-4537-9d95-0cf016fb1080"
}
```

Error for `404`:
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "Cannot create subscription. clusterId 0 is not supported."
}
```

##### `unsubscribe`
Request:
```http
POST /unsubscribe HTTP/1.1
Accept: application/json
Content-Type: application/json

{
  "subscriptionId": "9978c4f1-4465-4537-9d95-0cf016fb1080"
}
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "ok"
}
```

##### `getActiveSubscriptions`
Request:
```http
GET /subscriptions HTTP/1.1
Accept: application/json
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  "9978c4f1-4465-4537-9d95-0cf016fb1080",
  "a1b2c3d4-5678-90ab-cdef-1234567890ab"
]
```

### Message Storage

The `Message Storage API` SHOULD be used to retrieve information about messages that have been sent, 
received, or stored.

Message storage is populated by:
- Messages sent by the underlying node via the `Send API`.
- Messages received through an active subscription initiated by the `Subscribe API`.
- Messages fetched using the `History API`.

Messages can be identified by:
- The `message hash` as described in the [MESSAGE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/14/message.md#deterministic-message-hashing) specification.
- The `contentTopic` associated with the message, as defined in the [TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md#content-topic-format) specification.
- The `RequestID` described in the `Send API` section.
- The `SubscriptionID` described in the `Subscribe API` section.

These identifiers SHOULD be passed to the appropriate methods described below to retrieve or manage messages.

Depending on the presence of the `storeConfirmation` or `receiveConfirmation` options in the `Initial configuration`, the `Message Storage API` will trigger either the `Subscribe API` or `Acknowledgements` routine.

Once a message is added to the `Message Storage API`, or if any details about it change,
the appropriate events from the `message:*` event family (as described in the `Event Source API` section) MUST be invoked after the message is added or updated in the underlying storage.

#### Acknowledgements

Message Acknowledgement mechanism is a background operation that MUST:
- Perform store‑based reliability checks as specified in [P2P‑RELIABILITY](./p2p‑reliability.md).  
- Halt when the node’s health status transitions to `unhealthy` (as defined by the `Health API`).  
- Resume when the node’s health status transitions to either `minimally healthy` or `healthy`.  
- Set the `stored` or `received` field on `MessageRecord` once the message is acknowledged according to store‑based reliability.

This operation MUST be initiated when any of the following conditions is met:
- The `storeConfirmation` or `receiveConfirmation` property is `true` and `confirmContentTopics` is provided.  
- The `Subscribe API` is invoked for one or more content topics.  
- The `Send API` has successfully sent one or more messages.

For the recurring [STORE](../standards/core/store.md) queries it is RECOMMENDED to prioritize nodes in this order:
1. `staticStoreNode` specified in the initial configuration.
2. `preferredServiceNodes`, if provided.
3. Nodes discovered via network discovery.

#### Programmatic API

```typescript
type MessageRecord = {
  sending?: boolean;
  sent?: boolean;
  stored: boolean;
  received: boolean;
  requestId?: RequestID;
  message: IWakuMessage;
  error?: unknown;
};

interface MessageStorage {
  getByRequestId(id: RequestID): MessageRecord | undefined;
  getByHash(hash: string, skip?: number, take?: number): MessageRecord | undefined;
  getByContentTopic(contentTopic: string, skip?: number, take?: number): MessageRecord[];
  getBySubscriptionId(id: SubscriptionID, skip?: number, take?: number): MessageRecord[];
}
```

#### Properties of `MessageRecord`

##### `sending`
Defaults to `false`.
Indicates whether a message, originating from the underlying node,
is currently in the process of being transmitted via the `Send API`.
This property is set to `true` while the message is actively being sent or during retry attempts.
Once the send operation completes (either successfully or if retry attempts are halted),
the property MUST transition back to `false`.
This property is not applicable to messages that are received by the node.

##### `sent`
Defaults to `false`.
This field is applicable only to messages sent by the underlying node.
Indicates whether a message, sent by the underlying node via the `Send API`,
was successfully transmitted.
Once a message is sent successfully, the `sent` property becomes `true` and remains unchanged.
If the transmission fails, the property transitions to `false` and the `error` field is populated with implementation-specific details.
This update occurs only after the `sending` property has transitioned back to `false`.

##### `stored`
Defaults to `false`.
Indicates whether a message has been confirmed to be present in the underlying [STORE](../standards/core/store.md) protocol.
This property is populated based on a recurring [STORE](../standards/core/store.md) query described in `Acknowledgements` section or via the `History API`.
Once a message is confirmed as stored, the `stored` property MUST be updated to `true` as a final state.
If an error occurs during reception, the `error` field MUST be populated with implementation-specific details.

##### `received`
Defaults to `false`.
Indicates whether a message was successfully received from the network via `Subscribe API`.
Once a message is received, the `received` property MUST be updated to `true` as a final state.
If an error occurs during reception, the `error` field MUST be populated with implementation-specific details.

##### `requestId`
Defaults to `null`.
An optional property applicable only to messages sent by the underlying node via the `Send API`.
When present, it corresponds to the `RequestID` from the `Send API`, remains constant once set.

##### `message`
This includes, but is not limited to, the following attributes:
- `payload`: MUST contain the message data payload.
- `contentTopic`: MUST specify a string identifier used for content-based filtering.
- `meta`: If present, contains an arbitrary application-specific variable-length byte array (maximum 64 bytes) for supplementary details.
- `version`: If present, contains a version number to distinguish different types of payload encryption; if omitted, it SHOULD be interpreted as version 0.
- `timestamp`: If present, indicates the time the message was generated (in Unix epoch nanoseconds); if omitted, it SHOULD be interpreted as 0.
- `ephemeral`: If present and `true`, the message is considered transient and SHOULD not be persisted; if omitted or `false`, the message is considered non-ephemeral.

For a complete description of the message attributes,
refer to the [MESSAGE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/14/message.md#message-attributes) specification.

##### `error`
An optional property that, if present, indicates that an error occurred during the processing of the message—whether while sending or upon receiving.
This may include network, protocol, decoding errors, or other implementation-specific issues.
Once set, the error persists; however, if a message that initially encountered an error (e.g., associated with a `RequestID`) is later successfully transmitted, the error information MUST be cleared.

#### Methods

##### `getByRequestId`
Retrieves the `MessageRecord` corresponding to the specified `RequestID` associated with a send request for a particular message.
If the provided `RequestID` is invalid or no associated `MessageRecord` exists,
the method MUST return nothing.

##### `getByHash`
Retrieves the `MessageRecord` corresponding to the provided `message hash`.
The hash is computed deterministically based on the message content,
as described in the [MESSAGE](https://github.com/vacp2p/rfc-index/blob/8ee2a6d6b232838d83374c35e2413f84436ecf64/waku/standards/core/14/message.md#deterministic-message-hashing) specification.
If the provided hash is invalid or no associated `MessageRecord` exists,
the method MUST return nothing.

##### `getByContentTopic`
Retrieves an array of `MessageRecord` objects corresponding to messages that were circulated under the specified `contentTopic`.
The method accepts two optional parameters for pagination:
- `skip`: A non-negative integer specifying the number of initial matching records to bypass. If `skip` exceeds the total number of matching records, the method MUST return an empty array.
- `take`: A non-negative integer specifying the maximum number of records to return. If `take` is 0, the method MUST return an empty array. If `take` is greater than the number of available records, all remaining matching records are returned.

If both `skip` `and` take are omitted, the method MUST return all matching records.
If no records match the specified `contentTopic`, an empty array MUST be returned.

##### `getBySubscriptionId`
Retrieves an array of `MessageRecord` objects corresponding to messages received under the specified `SubscriptionID` (as defined in the `Subscribe API` section).
The method accepts two optional parameters for pagination:
- `skip`: A non-negative integer specifying the number of initial matching records to bypass. If `skip` exceeds the total number of matching records, the method MUST return an empty array.
- `take`: A non-negative integer specifying the maximum number of records to return. If `take` is 0, the method MUST return an empty array. If `take` is greater than the number of available records, all remaining matching records are returned.

If both `skip` and `take` are omitted, the method MUST return all matching records.
If no records match the specified `SubscriptionID`, an empty array MUST be returned.

#### REST API

##### By `RequestID`
Request:
```http
GET /message?requestId=xyz789 HTTP/1.1
Accept: application/json
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "sending": true,
  "sent": false,
  "stored": false,
  "received": false,
  "requestId": "xyz789",
  "message": {
    "payload": "U29tZSBvdGhlciBkYXRh",
    "contentTopic": "/example/another-topic"
  }
}
```

Error for `404`:
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "Message with requestId 'xyz789' not found"
}
```

##### By hash
Request:
```http
GET /message?hash=abc123 HTTP/1.1
Accept: application/json
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "sending": false,
  "sent": false,
  "stored": true,
  "received": false,
  "message": {
    "payload": "SGVsbG8gd29ybGQ=",
    "contentTopic": "/example/topic"
  }
}
```

Error for `404`:
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "Message with hash 'abc123' not found"
}
```

##### By `contentTopic`
Request:
```http
GET /messages?contentTopic=/messaging/2/api/utf8&skip=0&take=10 HTTP/1.1
Accept: application/json
```
Note: If skip and take are omitted, the request URL would be:
```http
GET /messages?contentTopic=/messaging/2/api/utf8 HTTP/1.1
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "sent": false,
    "stored": false,
    "received": true,
    "message": {
      "payload": "SW5pdGlhbCBtZXNzYWdl",
      "contentTopic": "/messaging/2/api/utf8"
    }
  }
]
```

Error for `404`:
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "No messages found for contentTopic '/messaging/2/api/utf8'"
}
```

##### By `subscriptionId`
Request:
```http
GET /messages?subscriptionId=1692854c-cbde-4afe-901f-1c86ecbd9ea1&skip=0&take=10 HTTP/1.1
Accept: application/json
```
Note: When omitting pagination parameters, the URL would be:
```http
GET /messages?subscriptionId=1692854c-cbde-4afe-901f-1c86ecbd9ea1 HTTP/1.1
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "sent": false,
    "stored": false,
    "received": true,
    "message": {
      "payload": "SW5pdGlhbCBtZXNzYWdl",
      "contentTopic": "/messaging/2/api/utf8"
    }
  }
]
```

Error for `404`:
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "No messages found for subscriptionId '1692854c-cbde-4afe-901f-1c86ecbd9ea1'"
}
```

### Health Indicator

The `Health Indicator API` SHOULD be used to monitor the operational health of a node operating under the `Messaging API`.
The criteria for determining a node’s health state MAY vary depending on the protocols in use and the node’s configuration.
Detailed criteria for each health state are provided in the subsequent sections.

For the purposes of this specification, three health states are defined:
- `unhealthy`: Indicates that the node has lost connectivity for message reception, sending, or both, and as a result, it cannot reliably process or transmit messages.
- `minimally healthy`: Indicates that the node meets the minimum operational requirements, although performance or reliability may be impacted.
- `healthy`: Indicates that the node is operating optimally, with full support for message processing and transmission.

Furthermore, any change in the node’s health state MUST trigger a `health:change` event via the `Event Source API` immediately after the state transition occurs.
Refer to `Event Source API` for more details.

#### Health states

##### `unhealthy`
Indicates that the node’s connectivity is insufficient for reliable operation.
In this state:
- No connections to service nodes are available, regardless of protocol.
- No peers are present in the [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) mesh.

##### `minimally healthy`
Indicates that the node meets the minimum connectivity requirements necessary to function,
although performance or reliability may be compromised.
In this state:
- The node has at least one connection to a service node implementing [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) and one connection to a service node implementing [LIGHTPUSH](../standards/core/lightpush.md).
- The [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) mesh includes connections to at least four other peers.

##### `healthy`
Indicates that the node’s connectivity meets or exceeds optimal operational thresholds.
In this state:
- The node has at least two connections to service nodes implementing [FILTER](https://github.com/vacp2p/rfc-index/blob/7b443c1aab627894e3f22f5adfbb93f4c4eac4f6/waku/standards/core/12/filter.md) and at least two connections to service nodes implementing [LIGHTPUSH](../standards/core/lightpush.md).
- The [RELAY](https://github.com/vacp2p/rfc-index/blob/0277fd0c4dbd907dfb2f0c28b6cde94a335e1fae/waku/standards/core/11/relay.md) mesh includes connections to at least six other peers.

##### Programmatic API

```typescript
enum HealthStatus {
  Unhealthy = "Unhealthy",
  MinimallyHealthy = "MinimallyHealthy",
  Healthy = "Healthy"
}

node.healthIndicator == "MinimallyHealthy";
```

##### REST API
Request:
```http
GET /health HTTP/1.1
Accept: application/json
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  status: "MinimallyHealthy"
}
```

### Event source

One component of the `Messaging API` is an event emitter that SHOULD be used to track various internal activities.
Consumers MAY subscribe to this event source to monitor message propagation,
network changes, or the health status of the node.
Events are dispatched only after an actual change to the underlying data occurs,
ensuring that notifications reflect genuine updates.

For accessing events via the REST API, long polling SHOULD be used.

#### Programmatic API

```typescript
type EventHandler = (event: CustomEvent<unknown>): void;

interface IEvents {
  public addEventListener(eventName: string, fn: EventHandler): void;
  public removeEventListener(eventName: string, fn: EventHandler): void;
}
```

##### `CustomEvent`
A container object that holds the payload of an event along with implementation-specific metadata.
The `detail` property carries the event payload, while other metadata may vary based on the Waku implementation.

##### `EventHandler`
A callback function that is invoked whenever an event with a supported `eventName` is dispatched.
Event handlers are triggered in the order in which they were added (i.e., in the order of subscription).
The function receives a `CustomEvent` containing the event payload.

##### `addEventListener`
Registers an event listener for a specified `eventName` using the provided `EventHandler`.
If the given `eventName` is not supported, the call MUST be silently ignored.
Additionally, if an exception is thrown within the function passed to `addEventListener`,
that error MUST be ignored and not affect subsequent event handling.

##### `removeEventListener`
Removes a previously registered event listener for the specified `eventName` and `EventHandler`.
If the `eventName` is not supported or the `EventHandler` was not previously registered,
the call MUST be silently ignored.

#### Events

##### `health:change`
This event MUST be emitted whenever the node’s health state changes,
as determined by the `Health Indicator API`.
The event’s payload (accessible via `event.detail`) is the new `HealthStatus` value (`Unhealthy`, `MinimallyHealthy`, or `Healthy`).
The event is dispatched immediately after the underlying data is updated to reflect the health state change.

Programmatic API:
```typescript
node.events.addEventListener("health:change", (event: CustomEvent<HealthStatus>) => {
  // event.detail holds the new HealthStatus value
  console.log(event.detail);
});
```

For accessing health change events via the REST API, long polling SHOULD be used.
Refer to `Health Indicator API` section for REST API endpoint.

##### `message:change`
This event MUST be dispatched whenever any of the key properties of a `MessageRecord` change.
Specifically `stored`, `received`,`sent` or `sending`.
The event payload (accessible via `event.detail`) is the updated `MessageRecord` reflecting the changes.

```typescript
node.events.addEventListener("message:change", (event: CustomEvent<MessageRecord>) => {
  // event.detail holds the updated MessageRecord
  console.log(event.detail);
});
```

For the `REST API`, long polling over the `Message Store API` SHOULD be used.

##### `message:sent`
This event MUST be dispatched immediately after a message that originated from the underlying node transitions from being in the `sending` state
(i.e., when `sending` becomes `false`) to a state where it is confirmed as successfully transmitted (i.e., `sent` becomes `true`).
The payload is the `MessageRecord` reflecting this updated status.

```typescript
node.events.addEventListener("message:sent", (event: CustomEvent<MessageRecord>) => {
  // event.detail holds the updated MessageRecord
  console.log(event.detail);
});
```

For the `REST API`, long polling over the `Message Store API` SHOULD be used.

##### `message:error`
This event MUST be dispatched when an error occurs during the processing of a message whether while sending or receiving.
In such cases, the `error` property of the corresponding `MessageRecord` is populated with implementation-specific error information.
The event payload is the `MessageRecord` containing the error details.
Once a previously errored message is successfully transmitted, the error information MUST be cleared and an updated event (such as `message:sent` or `message:change`) should be dispatched.

```typescript
node.events.addEventListener("message:error", (event: CustomEvent<MessageRecord>) => {
  // event.detail holds the updated MessageRecord
  console.log(event.detail);
});
```

For the `REST API`, long polling over the `Message Store API` SHOULD be used.

##### `message:received`
This event MUST be dispatched when a message is successfully received from the network via the `Subscribe API`.
The payload is the updated `MessageRecord`.

```typescript
node.events.addEventListener("message:received", (event: CustomEvent<MessageRecord>) => {
  // event.detail holds the updated MessageRecord
  console.log(event.detail);
});
```

For the `REST API`, long polling over the `Message Store API` SHOULD be used.

##### `message:stored`
This event MUST be dispatched when a message is confirmed to be `stored` in accordance with `Acknowledgements` section.
The payload is the updated `MessageRecord`.

```typescript
node.events.addEventListener("message:stored", (event: CustomEvent<MessageRecord>) => {
  // event.detail holds the updated MessageRecord
  console.log(event.detail);
});
```

For the `REST API`, long polling over the `Message Store API` SHOULD be used.

## Security/Privacy Considerations

See [WAKU2-ADVERSARIAL-MODELS](https://github.com/waku-org/specs/blob/master/informational/adversarial-models.md).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
