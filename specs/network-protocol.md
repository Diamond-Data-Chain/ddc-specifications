# Network Protocol

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

## Abstract

This document specifies the peer-to-peer networking layer of DDC nodes: transport, peer discovery, connection management, and the wire protocol for exchanging blocks, transactions, and consensus messages.

---

## Transport

DDC uses [libp2p](https://libp2p.io/) as its networking substrate.

### Supported Transports

| Transport | Usage |
|-----------|-------|
| TCP/IP | Primary transport for all environments |
| WebSocket over TLS | Browser and restricted-network environments |
| QUIC | Optional; improves performance in high-latency networks |

### Security

All connections MUST be encrypted using the [Noise Protocol Framework](https://noiseprotocol.org/) (`XX` handshake pattern) via libp2p Noise. Plaintext connections MUST NOT be accepted.

### Multiplexing

All connections MUST use [yamux](https://github.com/hashicorp/yamux) stream multiplexing, allowing multiple logical streams over a single TCP connection.

---

## Identity

Each node has a persistent **Peer Identity** derived from an Ed25519 keypair:

```
peer_id = Multihash(SHA-256(protobuf_encode(public_key)))
```

Peer IDs are represented as [Multiaddr](https://multiformats.io/multiaddr/) strings, for example:

```
/ip4/203.0.113.5/tcp/30333/p2p/12D3KooWBmAwcd4PJNJvfV89HwE48nwkRmAgo8Vy3uQEyNNHBox2
```

---

## Peer Discovery

### Bootstrap Nodes

New nodes MUST connect to at least one bootstrap node on startup. The bootstrap node list for each network is embedded in the node software and can be overridden by configuration.

**Mainnet bootstrap nodes:**

```
/dns4/boot1.ddc.network/tcp/30333/p2p/<peer-id>
/dns4/boot2.ddc.network/tcp/30333/p2p/<peer-id>
/dns4/boot3.ddc.network/tcp/30333/p2p/<peer-id>
```

### Kademlia DHT

After initial connection, nodes use [Kademlia DHT](https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf) to discover additional peers. Nodes MUST:

- Maintain a routing table of at least **25 peers**.
- Perform a DHT random walk every **5 minutes** to refresh routing tables.
- Publish their own peer record to the DHT every **60 minutes**.

### mDNS

Nodes MAY use mDNS for local peer discovery in development environments. mDNS MUST be disabled on mainnet nodes.

---

## Connection Management

### Target Peer Count

| Parameter | Value |
|-----------|-------|
| Minimum peers | 5 |
| Target peers | 25 |
| Maximum peers | 50 |

Nodes SHOULD aim to maintain connections to `target_peers`. If the peer count falls below `min_peers`, the node MUST aggressively attempt to find new peers via the DHT.

### Connection Limits

To prevent eclipse attacks, nodes MUST:

- Accept at most **10 inbound connections** from the same /24 IPv4 subnet (or /48 IPv6 subnet).
- Reject inbound connections that push the total above `max_peers`.

### Peer Scoring

Nodes maintain a score for each peer based on behavior:

| Event | Score Delta |
|-------|-------------|
| Valid block received | +5 |
| Valid transaction received | +1 |
| Invalid block received | -50 |
| Invalid transaction received | -10 |
| Timeout/disconnect | -5 |
| Equivocation evidence provided | +20 |

Peers with scores below **-100** MUST be disconnected and their `peer_id` banned for **1 hour**.

---

## Protocols (SubProtocols)

DDC uses libp2p protocol negotiation. Each sub-protocol is identified by a string path.

| Protocol ID | Purpose |
|-------------|---------|
| `/ddc/sync/1.0.0` | Block sync (request/response) |
| `/ddc/transactions/1.0.0` | Transaction gossip |
| `/ddc/consensus/1.0.0` | Consensus messages (validators only) |
| `/ddc/light/1.0.0` | Light client proof service |

### Protocol Negotiation

Protocols are negotiated using [multistream-select](https://github.com/multiformats/multistream-select). A node MUST reject connections that fail to negotiate any supported protocol.

---

## Block Sync Protocol (`/ddc/sync/1.0.0`)

Block sync uses a request-response pattern over a dedicated stream.

### Request

```
BlockSyncRequest {
    from_block: BlockNumber,
    max_blocks: u32,         // MUST NOT exceed 128
    direction:  SyncDirection,
}

SyncDirection {
    Ascending  = 0,
    Descending = 1,
}
```

### Response

```
BlockSyncResponse {
    blocks: Vec<Block>,
}
```

Nodes MUST respond to sync requests from peers with lower `from_block` than their current tip. Nodes MUST NOT send more than `max_blocks` blocks in a single response.

### Sync Strategy

On startup, nodes perform an **initial block download (IBD)**:

1. Request headers from the peer with the highest known block number.
2. Verify each header in sequence.
3. Download and validate block bodies in parallel from multiple peers.
4. Apply blocks to the state machine sequentially once validated.

---

## Transaction Gossip Protocol (`/ddc/transactions/1.0.0`)

Transactions are propagated using a gossip protocol with duplicate suppression.

### Transaction Announcement

```
TransactionAnnouncement {
    tx_hashes: Vec<Hash>,    // Max 256 hashes per announcement
}
```

Upon receiving an announcement, a node MUST request any unknown transactions from the sender.

### Transaction Request/Response

```
TransactionRequest {
    tx_hashes: Vec<Hash>,    // Max 64 per request
}

TransactionResponse {
    transactions: Vec<Extrinsic>,
}
```

### Gossip Rules

- A node MUST NOT re-announce a transaction to a peer that has already announced it.
- A node MUST NOT forward transactions that fail basic validity checks (signature, nonce format).
- A node MUST discard transactions that are more than **2 hours** old based on their `nonce` relative to the current chain state.

---

## Consensus Protocol (`/ddc/consensus/1.0.0`)

Consensus messages (Proposal, PrepareVote, CommitVote, ViewChange, NewView) are exchanged over this protocol. Only nodes in the active validator set SHOULD open streams on this protocol.

All messages are SCALE-encoded and prefixed with a 4-byte little-endian length field.

```
Message {
    length:  u32,
    payload: Bytes,   // SCALE-encoded consensus message
}
```

See [Consensus Protocol](consensus.md) for message definitions.

---

## Light Client Protocol (`/ddc/light/1.0.0`)

Full nodes MAY serve light client proof requests.

### Proof Request

```
LightProofRequest {
    block_hash:   Hash,
    storage_key:  Bytes,
}
```

### Proof Response

```
LightProofResponse {
    value:  Option<Bytes>,
    proof:  MerkleProof,
}
```

See [Data Model](data-model.md) for the `MerkleProof` structure.

---

## Security Considerations

- **Sybil Resistance:** Peer scoring and connection limits from the same subnet mitigate simple Sybil attacks. Validator participation requires bonded stake, providing economic Sybil resistance for consensus.
- **Eclipse Attacks:** Maintaining a diverse peer set (enforced by per-subnet limits) and using the Kademlia DHT reduce eclipse risk.
- **DoS:** Rate limiting MUST be applied per peer:
  - Maximum **100 block sync requests** per peer per minute.
  - Maximum **1000 transaction announcements** per peer per minute.
  - Peers exceeding limits MUST be disconnected and scored down.

---

## References

- [Architecture Overview](architecture.md)
- [Data Model](data-model.md)
- [Consensus Protocol](consensus.md)
- [libp2p Specification](https://github.com/libp2p/specs)
- [Noise Protocol Framework](https://noiseprotocol.org/)
- [Kademlia DHT](https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
