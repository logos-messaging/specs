---
title: WAKU2-OPERATOR-INCENTIVES
name: Waku Operator Incentives
category: Standards Track
tags: [waku/core-protocol]
editor: Alvaro R. <alrevuelta@status.im>
contributors:
---

## Abstract

This draft spec proposes how to economically incentivise node operators to participate in Waku RLN Relay, by distributing the rewards paid by users when registering their RLN memberships.
It is assumed that users pay recurrently for their RLN membership. The generated fees are distributed evenly to the operators participating in each epoch.
Only registered operators that put a bond at stake are eligible for such rewards. And only the nodes that vote on the correct Merkle root get the reward.

It also introduces on-chain consensus. All operators agree on a Merkle root representing all the messages relayed during each epoch.
This enables use cases where nodes can prove that a message was indeed sent and is part of the waku message set.

## Syntax

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Background

This draft spec comes from [here](https://github.com/waku-org/research/issues/101).

The waku ecosystem contains two main economic agents:
* Users: Make use of the network to send messages. They hold a RLN membership with some associated cost. They are the consumers.
* Operators: Run the network infra, providing services to users in the network. They are the producers.

This proposal connects them together by organising in a decentralized way how users and operators can interact with each other via a peer-to-peer network, settling
all decisions in a third-party blockchain, making use of smart contracts.

Definitions:
* Operator: Entity running `nwaku` that has deposited a stake and is eligible for root voting.
* Stake: Collateral posted by each operator, entitling it to participate in consensus.
* Slash: Penalty applied to a node operator, subtracted from its stake.
* node vs operator. one holds a stake, the other not.
* RLN contract: EVM compatible smart contract used to settle between operators and users.
* Consensus: Onchain agreement on a given Merkle root.


## Specification

This spec covers the following:
* The honest operator
* The lazy operator
* Node penalties
* Root consensus

### The honest operator

An operator wanting to participate in consensus to be eligible for rewards SHALL register in the `RLN_CONTRACT` staking `STAKE_AMOUNT`.
This entitles it to participate in the Merkle root voting every `RLN_EPOCH_SIZE_SECONDS`.
Both voting and staking SHALL be done in a public EVM-compatible blockchain.

In order to prevent other operators from taking over the network, an operator SHALL wait an entry queue to start being eligible.
The contract SHALL process `OPERATORS_ENTER_EPOCH` new operators per `RLN_EPOCH_SIZE_SECONDS`.

In order for the network to kick-off, a minimum of `MIN_GENESIS_OPERATOR_COUNT` SHALL exist before the network can start.
This prevents the network from kicking off with a small set of operators.

An operator wanting to stop participating MAY withdraw its stake at any time. Only `OPERATORS_EXIT_EPOCH` operators can exit every `RLN_EPOCH_SIZE_SECONDS`.
This prevents many operators from exiting at once, putting the network consensus at risk.

Once part of the network, the honest operator is eligible for rewards in every epoch, and SHALL:
* Participate in the network by relaying messages to other peers, constructing a local Merkle root of the messages seen during a given epoch.
* Reply to other nodes' challenges (see The Challenge), proving it's an honest node. This ensures that nodes are not just following the root but also storing the messages.
* Use store-sync to ensure it's not missing past messages. We acknowledge that nodes may have some downtime, so it's accepted that some of these messages will come from store and not relay.

After each epoch, the Merkle root agreed by a supermajority will get consolidated, and the operators that voted for it will get rewarded proportionally to the generated fees.
The honest operator is incentivized to get all messages relayed in the network since that will allow it to know the Merkle root that other operators know as well.
The honest operator is also incentivised to relay messages to other peers, since that will improve its gossipsub score, making it more likely to have a rich set of connected peers and not miss any message.

Every time an operator votes on a root agreed by the supermajority, its balance `CLAIMABLE_BALANCE` SHALL be increased proportionally to the amount of nodes that 
agreed on the root and the generated RLN fees during that epoch.

### The lazy operator

A lazy operator is defined as an operator not doing the required work but trying to get the reward. There are two lazy behaviors:
* Sniff the blockchain to get other operators Merkle roots, and copy them.
* Don't run relay nor store messages and use store-sync to construct a valid Merkle root.

These lazy behaviors are fixed with:
* A commit-reveal scheme for voting: It makes the voting secret.
* A gossipsub challenge: Nodes are challenged to prove they know the content of random message hashes.
If the challenge is not responded successfully, the peer gets de-scored and eventually disconnected.


For the commit-reveal, registered operators vote every `RLN_EPOCH_SIZE_SECONDS` using the `RLN_CONTRACT` to the Merkle root they believe represents all the messages accumulated within that epoch.
Once the epoch is due, a `COMMIT_WINDOW_SLOTS` SHALL start, where operators commit to a `H(root + salt)` they consider to represent all messages seen during that epoch.
Note that the `salt` is used to prevent a lazy node from copying both the commitment and the reveal. It MAY be the operator's address.

Once `COMMIT_WINDOW_SLOTS` is due, no more votes SHALL be accepted for that epoch, and a `REVEAL_WINDOW_SLOTS` SHALL start.
During this window node operators SHALL reveal their secret `root`.
If it matches the previous commitment `hash(root + salt)`, then their vote SHALL be accepted.

Finally, once `REVEAL_WINDOW_SLOTS` is due eligible node operators can claim their share of the reward.
Their `CLAIMABLE_BALANCE` SHALL be increased and MAY be claimed via the `RLN_CONTRACT` at any time.

The gossipsub challenge SHALL use gossipsub's P5 "Application-Specific" score.
The P5 score SHALL increase when the challenge is answered correctly and decreases when it is not.
It SHALL be designed in a way where few occasional invalid responses don't affect much the score and decay over time to a point where it no longer matters.
When the node fails multiple challenges in a row it SHALL lower its score to a point where a disconnection is triggered.

The challenge SHALL be done every `CHALLENGE_TIME_SEC` to every peer it's connected.
It SHALL verify that other peers know the content of a message given its hash.
To do so each node SHALL pick a random message within the `RLN_EPOCH_SIZE_SECONDS` of the current epoch.
A challenger SHALL request a hash `H` and include a `salt` in its request.
A honest responder SHALL reply with `H(m || salt)`. If they match, the responder is proving the knowledge of the message content.
The challenger SHALL increase the score of the responder if it matches the local copy, and lower it otherwise.
The challenge SHALL leave a window of `PROPAGATION_TIME_SECONDS` so that no messages between `now_timestamp-PROPAGATION_TIME_SECONDS, now_timestamp` are used for the challenge.
This ensures that the challenge doesn't run with messages that may not have been propagated yet.


### Node penalties

If an operator does not submit a vote for a given epoch, it SHALL lose `INACTIVE_EPOCH_PENALTY` from its `STAKE_AMOUNT` but can continue participating in the protocol.
If an operator does not submit votes for `NUM_BAN_EPOCHS` consecutive epochs, it SHALL be kicked out of the protocol with an `x*INACTIVE_EPOCH_PENALTY` penalty `<STAKE_AMOUNT`.
The remaining stake SHALL be returned.

### Root consensus

After `REVEAL_WINDOW_SLOTS` the commit-reveal phase is done and votes to each root SHALL be counted.
If any root `CONSENSUS_ROOT` gets more than `SUPERMAJORITY_PERCENT` votes, it SHALL be considered as the valid Merkle root for that epoch.

Any operator that voted to `CONSENSUS_ROOT` SHALL be eligible for rewards, computed proportionally given the number of operators participating in consensus.

If no consensus is reached during the first round, a new round SHALL start until a consensus of `SUPERMAJORITY_PERCENT` is reached.
In theory, infinite rounds are possible, but given the following, it SHALL eventually converge.
* Nodes can verify that a waku message is valid (on-chain root and RLN proof).
* Nodes can sync with each other via store sync.
* Once the epoch is due no new messages are accepted for that epoch.
* Peers form a mesh with a low probability of network partitions (discv5 + gossipsub).

If consensus is not reached during the first round, an operator MAY opt to:
* Trigger a more aggressive peer discovery.
* Temporally allow more connected peers.
* Trigger a more aggressive store sync.

This should reduce the probability of a network partition and the node missing messages.
