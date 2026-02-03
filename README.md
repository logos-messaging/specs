# Waku Specifications

Waku builds a family of privacy-preserving, censorship-resistant communication protocols for web3 applications.
This repository contains specifications for the Waku suite of protocols.

## List of Specifications:

- Standards
  - Core: Standards and protocols for the core Waku p2p communications offering.
  - Application: Standards and protocols that describe various applications or encryption use cases built on top of a Waku network.
- Informational: Waku design issues, general guidelines or background information that does not constitute a new feature.

### Core standards

| Waku Specifications | Description |
| ---- | -------------- |
|[10/WAKU2](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/10/waku2.md)| Waku Overview |
|[11/WAKU2-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/11/relay.md)| Waku Relay |
|[12/WAKU2-FILTER](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/12/filter.md)| Waku Filter |
|[13/WAKU2-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/13/store.md)| Waku Store |
|[14/WAKU2-MESSAGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/14/message.md)| Waku Message |
|[17/WAKU2-RLN-RELAY](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/17/rln-relay.md)| Waku RLN Relay |
|[19/WAKU2-LIGHTPUSH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/19/lightpush.md)| Waku Lightpush |
|[33/WAKU2-DISCV5](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/33/discv5.md)| Waku Discovery v5 Ambient Peer Discovery |
|[34/WAKU2-PEER-EXCHANGE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/34/peer-exchange.md#abstract)| Waku Peer Exchange |
|[36/WAKU2-BINDINGS-API](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/36/bindings-api.md)| Waku C Bindings API |
|[64/WAKU2-NETWORK](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/64/network.md)| Waku Network |
|[66/WAKU2-METADATA](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/core/66/metadata.md)| Waku Metadata |
|[WAKU2-ENR](standards/core/enr.md)| Waku Usage of ENR |
|[WAKU2-INCENTIVIZATION](standards/core/incentivization.md)| Waku Incentivization |
|[WAKU2-RLN-CONTRACT](standards/core/rln-contract.md) | Waku RLN Contract Specification |
|[RELAY-SHARDING](standards/core/relay-sharding.md)| Waku Relay Sharding |
|[WAKU2-SYNC](standards/core/sync.md) | Waku Sync |
|[WAKU-MIX](standards/core/mix.md) | Waku Mix |
|[WAKU-RENDEZVOUS](standards/core/rendezvous.md) | Waku Rendezvous |

### Application standards

| Waku Specifications | Description |
| ---- | -------------- |
|[20/TOY-ETH-PM](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/20/toy-eth-pm.md)| Toy Ethereum Private Message |
|[21/WAKU2-FAULT-TOLERANT-STORE](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/21/fault-tolerant-store.md)| Waku Fault-Tolerant Store |
|[22/TOY-CHAT](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/22/toy-chat.md)| Waku Toy Chat |
|[26/WAKU2-PAYLOAD](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/26/payload.md)| Waku Message Payload Encryption |
|[53/WAKU2-X3DH](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/53/x3dh.md)| X3DH Usage for Waku Payload Encryption |
|[54/WAKU2-X3DH-SESSIONS](https://github.com/vacp2p/rfc-index/blob/main/waku/standards/application/54/x3dh-sessions.md)| Session Management for Waku X3DH |
|[DANDELION](standards/application/dandelion.md)| Waku Dandelion |
|[WAKU2-DEVICE-PAIRING](standards/application/device-pairing.md)| Device Pairing and Secure Transfers with Noise |
|[WAKU2-NOISE](standards/application/noise.md)| Noise Protocols for Waku Payload Encryption |
|[TOR-PUSH](standards/application/tor-push.md)| Waku Tor Push |
|[RLN-KEYSTORE](standards/application/rln-keystore.md)| Waku RLN Keystore JSON schema |
|[P2P-RELIABILITY](standards/application/p2p-reliability.md)| Waku P2P Reliability |
|[WAKU-API](standards/application/waku-api.md)| Waku API |

### Informational

| Waku Specifications | Description |
| ---- | -------------- |
|[23/WAKU2-TOPICS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/23/topics.md)| Waku Topic Usage Recommendations |
|[27/WAKU2-PEERS](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/27/peers.md)| Waku Peer Management Recommendations |
|[29/WAKU2-CONFIG](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/29/config.md)| Waku Parameter Configuration Recommendations |
|[30/ADAPTIVE-NODES](https://github.com/vacp2p/rfc-index/blob/main/waku/informational/30/adaptive-nodes.md)| Adaptive Nodes |
|[ADVERSARIAL-MODELS](informational/adversarial-models.md)| Waku Adversarial Models and Attack-based Threat List |
|[RELAY-STATIC-SHARD-ALLOC](informational/relay-static-shard-alloc.md)| Waku Relay Static Shard Allocation |

## Resources
Relevant Waku resources related to the specifications located in this repository:
- [Waku.org](https://waku.org/)
- [nwaku: Waku Node](https://github.com/waku-org/nwaku)

## Contributions 

Contributions are welcome from any party. 
Contributors can create specifications relating to the Waku domain and
create a pull request to begin discussion.

Please adhere to the following contribution guidelines:
- use the recommended [template](./template.md) for new proposed specifications
- use keywords as per the language recommendations in the [template](./template.md) and [Vac COSS](https://github.com/vacp2p/rfc-index/blob/a5b24ac0a27da361312260f9da372a0e6e812212/vac/1/coss.md#language)
- use [semantic breaks](https://sembr.org/)
- links to Waku, Vac or other [IFT](https://free.technology/)-related specifications must be to the corresponding Github repository and not to a webpage.
For example, Waku specs reside in [waku-org/specs](https://github.com/waku-org/specs) and Vac RFCs in [vacp2p/rfc-index](https://github.com/vacp2p/rfc-index/).
- we no longer use "v2" in the _name_ of new specifications (and existing spec names inconsistent with this rule will eventually be revised). For example, do not call your specification "Waku v2 New Protocol" but simply "Waku New Protocol"
- the _title_ of new specifications must be prefixed with `WAKU2-` to differentiate it from other projects' specs and previous RFC generations. For example, "Waku New Protocol" could be titled `WAKU2-NEW-PROTOCOL`.

New specifications are considered a proof of concept.
Once a rough consensus is reached towards stabilization, 
the specification may be considered to receive the draft status and 
further discussion will continue on the [Vac RFC-Index](https://github.com/vacp2p/rfc-index) repository.

**NOTE:** Specifications located in this repository should be considered not production ready.
Discussion should be conducted with the intention of maturing each specification.

Head over to the [Vac RFC-Index](https://github.com/vacp2p/rfc-index) repository where other Waku specifications live.
