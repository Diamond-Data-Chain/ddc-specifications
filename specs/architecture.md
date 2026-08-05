# Architecture Overview

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

## Abstract

This document describes the high-level architecture of the Diamond Data Chain (DDC) ecosystem: its components, their responsibilities, and the design principles that guide the system.

---

## Design Principles

1. **Verifiability** – Every data record MUST be independently verifiable by any participant without trusting a central authority.
2. **Immutability** – Once a record is finalized, it MUST NOT be altered or deleted except through a governance-approved migration process.
3. **Openness** – The protocol MUST be open-source and permissionless to read. Write access is governed by staking and identity requirements.
4. **Interoperability** – DDC nodes MUST expose standard interfaces to integrate with external systems (ERP, blockchain bridges, certification bodies).
5. **Auditability** – All state transitions MUST be recorded with cryptographic proofs sufficient to reconstruct any historical state.
6. **Resilience** – The network MUST tolerate the failure of up to one-third of validator nodes without loss of liveness or safety.

---

## System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        DDC Ecosystem                            │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│  │  Client Apps │   │   Indexers   │   │  Bridge Adapters │    │
│  └──────┬───────┘   └──────┬───────┘   └────────┬─────────┘    │
│         │                  │                    │               │
│  ┌──────▼──────────────────▼────────────────────▼─────────┐    │
│  │                     Node API Layer                      │    │
│  │              (JSON-RPC 2.0  /  REST HTTP)               │    │
│  └──────────────────────────┬────────────────────────────-─┘    │
│                             │                                   │
│  ┌──────────────────────────▼────────────────────────────-─┐    │
│  │                   DDC Full Node                          │    │
│  │                                                          │    │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │    │
│  │  │  Mempool    │  │  Consensus   │  │  State Machine │  │    │
│  │  │  (pending   │  │  Engine      │  │  (EVM-compat.) │  │    │
│  │  │   txns)     │  │  (DDC-BFT)   │  │                │  │    │
│  │  └──────┬──────┘  └──────┬───────┘  └───────┬────────┘  │    │
│  │         │                │                  │            │    │
│  │  ┌──────▼────────────────▼──────────────────▼────────┐  │    │
│  │  │                   Block Store                      │  │    │
│  │  │        (append-only, Merkle-indexed)               │  │    │
│  │  └────────────────────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           Peer-to-Peer Network Layer (libp2p)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1 DDC Full Node

A **Full Node** is the primary participant in the DDC network. It:

- Maintains a complete copy of the DDC blockchain from genesis.
- Validates and propagates transactions and blocks.
- Participates in consensus if configured as a validator node.
- Exposes the Node API to clients.

Full Nodes MUST implement all normative requirements in this specification set.

### 2.2 Validator Node

A **Validator Node** is a Full Node that has bonded sufficient stake to participate in the DDC-BFT consensus protocol. Validators:

- Propose and vote on blocks.
- Sign attestations over finalized blocks.
- Are subject to slashing for Byzantine behavior.

See [Consensus Protocol](consensus.md) for details.

### 2.3 Light Node

A **Light Node** downloads only block headers and uses Merkle proofs to verify specific state entries without storing the full chain. Light Nodes:

- MUST verify block headers against the validator set commitment.
- MUST request Merkle inclusion proofs for any data they rely on.
- MUST NOT participate in consensus or block production.

### 2.4 Node API Layer

The Node API exposes chain state and transaction submission to external clients. It supports:

- **JSON-RPC 2.0** over HTTP and WebSocket.
- **REST/HTTP** for resource-oriented queries.

See [Node API](node-api.md) for the full specification.

### 2.5 Indexers

Indexers are off-chain services that subscribe to DDC events and build queryable databases for specific use cases (e.g., diamond provenance lookup). Indexers are not part of the core protocol but MUST use only verified data sourced through the Node API.

### 2.6 Bridge Adapters

Bridge Adapters connect DDC to external blockchains (e.g., Ethereum, Polkadot) or legacy databases. They MUST use the canonical DDC data encoding (see [Data Model](data-model.md)) and MUST verify DDC Merkle proofs before relaying data.

---

## Network Topology

DDC uses a fully peer-to-peer topology with no privileged central nodes. Nodes discover each other through:

1. **Bootstrap Nodes** – A hard-coded list of well-known bootstrap nodes is included in the node software. Bootstrap nodes are maintained by the DDC Foundation but MUST NOT have special protocol privileges.
2. **mDNS** – Local network discovery for development and test environments.
3. **Kademlia DHT** – Ongoing peer discovery in the live network.

---

## Data Flow

### Transaction Submission

```
Client ──POST /tx──► Full Node Mempool
                          │
                     Validation
                          │
              ┌───────────▼──────────┐
              │  Broadcast to peers  │
              └───────────┬──────────┘
                          │
                    Block Proposal
                          │
                    DDC-BFT Vote
                          │
                    Finalization
                          │
                    State Update
```

### Data Query

```
Client ──GET /record/{id}──► Full Node
                                  │
                            Block Store
                                  │
                         Merkle Proof Generation
                                  │
                    ◄──── Record + Proof ──────────
```

---

## Security Model

DDC's security rests on the following assumptions:

- **Honest-Majority Stake** – The protocol is safe as long as fewer than one-third of bonded stake is controlled by Byzantine validators.
- **Cryptographic Primitives** – The protocol relies on Ed25519 for signatures, SHA-256 for hashing, and BLS12-381 for aggregate signatures.
- **Economic Incentives** – Validators risk losing bonded stake (slashing) for equivocation or persistent unavailability.

---

## Upgrade Path

Protocol upgrades follow the governance process defined in [Governance](governance.md). Hard forks require:

1. A passed governance proposal specifying the target block height.
2. Node software upgrade before the activation block.
3. An updated genesis commitment included in the new software release.

---

## References

- [Data Model](data-model.md)
- [Consensus Protocol](consensus.md)
- [Network Protocol](network-protocol.md)
- [Node API](node-api.md)
- [Node Operations](node-operations.md)
- [Governance](governance.md)
- [libp2p Specification](https://github.com/libp2p/specs)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
