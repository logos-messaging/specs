---
title: CONTENTFRAME
name: 
category: Standards Track
tags: 
editor: Jazzz
contributors:
---

## Abstract


## Background / Rationale / Motivation



## Theory / Semantics

### Definitions

- **Type:** A data structure used to store data
- **ContentType:** A `type` used specifically encode data intended for applications

### ContentFrame

A ContentFrame is used to distinguish the application level content, from payloads intended to modify protocol state -- while also providing the necessary context to properly decode it.

This specification provide:
- a structured approach to enable inter-operation of content types between applications.
- Flexible mechanism that allows permissionless type definition by all developers.


### Domain

A domain defines the authority which governs a set of content types. It cannot be assumed that all specifications covering content types reside in the same location or are governed by the same entity. By including the `domain` receiving application developers can determine where the definition of the type resides, regardless of where it originated.

- a domain MUST be a valid url.
- a domain SHOULD provide specifications for all of its defined types.

To reduce payload size, Domains are mapped to an integer `domain_id`. The mappings can be found [here](#appendix-a-domains)

- a domain_id MUST be an integer value in the range [0, 65,535]
- a domain_id MUST correspond to a single unique domain

### Tag

A tag uniquely defines a type within a domain. Domains 

- a tag MUST correspond to a single unique type within a domain

Each domain is responsible for managing its tags and providing documentation to developers on how the corresponding types are used.


## Wire Format Specification / Syntax

```protobuf
message ContentFrame {
    uint32 domain_id = 1;
    uint32 tag = 2;
    bytes bytes = 3;
}
```

**domain_id:** This field contains an integer which identifies the domain of this type.
**tag:** This field contains an integer which identifies which type `bytes` contains
**bytes:** This field contains the encoded payload 



## Implementation Suggestions (optional)

### Tags -> Specifications

If possible the integer tag values should be the same as the specification which defines the type used. While not necessary, using the specification id directly removes the requirement to maintain a separate mapping of tag -> specification. 

### Fragmentation

This protocol allows for multiple competing definitions of similar content types. Having multiple definitions of a `TextMessage` or `Image` will increase fragmentation between applications. Where possible reusing existing types will reduce burden on app developers, and increase interoperability between apps.

Domains should focus on providing types unique to their service or usecase.


## Security/Privacy Considerations




# Appendix A: Domains
**TODO:** 
- Find appropriate home for this.

Domain ID's are provided on a first come first serve basis.


- A domain MUST only appear once in the table
- 

| domain_id |  specification repository |
|-----------|-------------------------------------|
| 0         |   https://github.com/waku-org/specs |   