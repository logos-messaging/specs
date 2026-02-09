---
title: Introduction Bundle Encoding
name: introduction-bundle-encoding
category: Standards Track
tags: core, encoding
editor: Patryk <patryk@status.im>
contributors: Jazz <jazz@status.im>
---

## Abstract

This specification defines the encoding format for Introduction Bundles — the
out-of-band information shared so that a remote party can initiate contact with
the bundle publisher.

## Background / Rationale / Motivation

Users need a way to share contact information across arbitrary channels
(messaging apps, emails, QR codes, URLs, CLI terminals) without relying on a
centralized directory. The Introduction Bundle provides the cryptographic
material required to establish an encrypted conversation.

The encoding must be:
- Copy-paste safe across arbitrary text transports.
- Space-efficient for manual sharing.
- Version-aware to support protocol evolution.

## Theory / Semantics

An Introduction Bundle is bound to a specific protocol version; its encoded
form identifies the version unambiguously.

### Why ASCII, Not Unicode

The encoded string consists entirely of printable ASCII. This is a natural
consequence of the design rather than a defensive choice against Unicode:

1. **No human-readable content.** The format contains a fixed prefix, a numeric
   version, and a binary-to-text encoded payload. These components do not
   require characters outside ASCII. Common binary-to-text encodings such as
   hex, base32, and base64 produce ASCII output.
2. **Encoding stability.** Printable ASCII characters are represented
   identically in widely deployed text encodings such as UTF-8 and Latin-1,
   avoiding ambiguity in character interpretation across transports.
3. **Restricted visible alphabet.** Limiting the character set reduces the risk
   of visually confusable or non-rendering characters during manual comparison,
   transcription, or copy-paste across different platforms, terminals, and
   fonts.

### Delimiter Choice

The `_` character is present in the alphabets of common binary-to-text
encodings (hex, base32, base64url) and may therefore appear inside the payload.
Parsers MUST split the encoded string on the **first three** `_` characters;
everything after the third `_` is the payload verbatim.

The underscore was chosen because it is a printable ASCII character that is
safe in URLs, filenames, and plain text without requiring escaping. Delimiters
outside this category (e.g., `.` or `:`) may require percent-encoding in
certain URL contexts, undermining the transport-safety goal.

## Wire Format Specification / Syntax

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC 2119].

### Text Format

The Introduction Bundle is encoded as a single ASCII string:

```
logos_<namespace>_<version>_<payload>
```

| Field       | Description                                                       |
|-------------|-------------------------------------------------------------------|
| `logos`     | Fixed prefix. Identifies the string as a Logos protocol artifact. |
| `namespace` | Domain identifier. Distinguishes bundle types.                    |
| `version`   | Protocol version number. Decimal integer, no leading zeros.       |
| `payload`   | Binary-to-text encoded payload (version-specific).                |

Fields are separated by `_` (underscore).

### V1

#### Parameters

| Parameter   | Value                          |
|-------------|--------------------------------|
| `namespace` | `chatintro`                    |
| `version`   | `1`                            |
| `encoding`  | base64url, no padding          |
| `payload`   | Protobuf-encoded `IntroBundle` |

#### Payload Encoding

V1 uses **base64url without padding** ([RFC 4648 §5], padding
characters `=` omitted). This encoding was chosen for:

- URL safety without percent-encoding.
- ~33% size overhead (compared to ~100% for hex).
- Wide library support across languages.

Future versions MAY choose a different binary-to-text encoding.

#### Binary Payload

The payload is a Protocol Buffers (proto3) encoding of:

```proto
syntax = "proto3";

message IntroBundle {
  bytes installation_pubkey = 1;  // 32 bytes, X25519
  bytes ephemeral_pubkey    = 2;  // 32 bytes, X25519
  bytes signature           = 3;  // 64 bytes, XEdDSA
}
```

The encoding MUST use standard proto3 serialization. Canonical (deterministic)
serialization is NOT REQUIRED; decoders MUST accept any valid proto3 encoding
of the message.

## Implementation Suggestions

### V1

#### Encoding Steps

1. Construct the `IntroBundle` protobuf message.
2. Serialize using proto3 encoding (~134 bytes).
3. Encode as base64url without padding (~179 characters).
4. Prepend the preamble with version prefix.

The resulting string is ~197 printable ASCII characters.

## Security/Privacy Considerations

The Introduction Bundle contains long-term public key material. Publishing it
reveals the installation's public identity to anyone who receives the bundle.

The signature prevents tampering but does not provide confidentiality. Bundles
should be transmitted over channels appropriate for the user's threat model.

## Copyright

Copyright and related rights waived via [CC0].

## References

- [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) — Key words for use in RFCs
- [RFC 4648 §5](https://www.rfc-editor.org/rfc/rfc4648#section-5) — Base 64 Encoding with URL and Filename Safe Alphabet
- [CC0](https://creativecommons.org/publicdomain/zero/1.0/) — Creative Commons Zero Public Domain Dedication

[RFC 2119]: https://www.ietf.org/rfc/rfc2119.txt
[RFC 4648 §5]: https://www.rfc-editor.org/rfc/rfc4648#section-5
[CC0]: https://creativecommons.org/publicdomain/zero/1.0/
