---
title: CHAT-CAST
name: Roles and Entities Used in Chat Protocol Documentation
status: raw
category: Informational
tags: [chat/informational]
editor: Jazzz <jazz@status.im>
contributors:
---

## Abstract

This document defines a reusable cast of characters to be used in chat protocol documentation and related supporting materials.
The goal is to improve clarity and consistency when describing protocol roles, actors, and message flows.

## Background

When documenting applications and protocols, it is often beneficial to define a consistent set of characters representing common roles.
A shared cast allows authors to convey meaning and intent quickly to readers.
Readers are not required to understand these meanings, however consistent usage can make comprehension faster to achieve. 

This approach is well established in cryptographic literature, where `Alice` and `Bob` are commonly used to describe participants in key exchange protocols.
Within that context, Alice typically initiates the exchange and Bob responds.
Readers familiar with this convention can quickly understand protocol flows without additional explanation.

In messaging and communication protocols, a similar approach can be helpful, particularly when describing multiple actors and roles required for correct protocol operation.
However, reusing `Alice` and `Bob` in these contexts can introduce ambiguity:

- In cryptography, Alice is the initiator of a key exchange, but in a communication protocol the initiator role may vary by sub-protocol or phase.
- Complex, multi-step systems may embed multiple cryptographic and application-level processes, each with their own initiator and responder.
- The use of Alice and Bob implicitly frames the discussion as cryptographic, which may be misleading when describing application-level behavior such as message encoding, routing, or reliability.

For these reasons, when documenting communication protocols that integrate multiple roles and procedures, it is preferable to use a context-specific cast of characters designed for that domain.

## Guidelines

### Use of Alice and Bob

`Alice` and `Bob` SHOULD be used when describing novel cryptographic constructions or key exchange mechanisms that are not embedded within higher-layer communication protocols.
These names are widely understood in cryptographic contexts, and introducing alternatives would reduce clarity.

Communication protocols may incorporate cryptographic components, but they are not themselves cryptographic key exchanges.
When documenting application-centric or protocol-level processes, the cast defined in this document SHOULD be used instead.
This separation establishes clear contextual boundaries and prepares the reader to reason about different layers independently.

### Standalone Documents

Knowledge of this cast MUST NOT be a requirement to understand a given document.
Documents MUST be fully standalone and understandable to first-time readers.

### Consistency

Use of the cast is optional.
Characters SHOULD only be used when their presence improves understanding.
Using characters in the wrong context can negatively impact comprehension by implying incorrect information.
It is always acceptable to use other identifiers.

## Character List

The following characters are defined for use throughout the documentation of chat protocols. Documentation and examples focus on a portion of a real clients operation for simplicity. Using the character who corresponds to the role or perspective being highlighted, can help convey this information to readers.

**Saro**  
Sender ::
Saro is the participant who sends the first message within a given time window or protocol context.
Saro MAY be the party who initiates a conversation, or simply the first participant to act relative to a defined starting reference.

**Raya**  
Recipient ::
Raya is the participant who receives the first message sent by Saro.
After the initial exchange, Raya MAY send messages and behave as a regular participant in the conversation.
When documenting message receipt or processing, Raya’s perspective SHOULD be used.

**Pax**  
Participant ::
Pax represents an additional member of a conversation, typically in a group context.
Pax is often used when the specific identity or perspective of the participant is not relevant to the discussion.

## Decision Criteria

The following criteria SHOULD be applied when considering the introduction of new character names or roles.

### Clarity

Names without strong pre-existing associations or implied behavior SHOULD be preferred where possible.

### Brevity

Short, easily distinguishable names SHOULD be preferred, provided they do not reduce clarity.

### Inclusivity

The cast of characters SHOULD be diverse, culturally neutral, and avoid reinforcing stereotypes.

### Mnemonic Naming 

Where possible the characters name should hint at their role in order to make them easier to remember.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

[A global cast of characters for cryptography](https://github.com/jhaaaa/alix-and-bo)