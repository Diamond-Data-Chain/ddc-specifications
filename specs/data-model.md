# Data Model

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

## Abstract

This document defines the canonical data structures, schemas, and encoding formats used throughout the DDC protocol. All DDC implementations MUST use the structures and encodings defined here for any data that is stored on-chain, included in blocks, or transmitted over the network protocol.

---

## Encoding

### Primary Encoding: SCALE

All on-chain and network data MUST be encoded using [SCALE (Simple Concatenated Aggregate Little-Endian) codec](https://docs.substrate.io/reference/scale-codec/). SCALE is a compact, deterministic, and schema-free binary codec.

### JSON Encoding

Human-readable representations (e.g., in API responses) use JSON. JSON field names MUST match the field names specified in this document. Numeric values that exceed 53 bits MUST be represented as decimal strings to avoid IEEE 754 precision loss.

### Hex Encoding

Binary fields (hashes, public keys, signatures) in JSON MUST be encoded as lowercase hexadecimal strings prefixed with `0x`.

---

## Primitive Types

| Type | Size | Description |
|------|------|-------------|
| `u8` | 1 byte | Unsigned 8-bit integer |
| `u32` | 4 bytes | Unsigned 32-bit integer (little-endian) |
| `u64` | 8 bytes | Unsigned 64-bit integer (little-endian) |
| `u128` | 16 bytes | Unsigned 128-bit integer (little-endian) |
| `Hash` | 32 bytes | SHA-256 digest |
| `PublicKey` | 32 bytes | Ed25519 public key |
| `Signature` | 64 bytes | Ed25519 signature |
| `AggregateSignature` | 96 bytes | BLS12-381 G2 signature |
| `BlockNumber` | `u64` | Sequential block height, 0-indexed from genesis |
| `Timestamp` | `u64` | Unix timestamp in milliseconds (UTC) |
| `AccountId` | 32 bytes | Blake2b-256 hash of a public key |
| `Balance` | `u128` | Token balance in smallest unit (pico-DDC, 10⁻¹²) |

---

## Core Structures

### Block Header

The block header is the fundamental unit of chain continuity. Every block MUST contain a valid header.

```
BlockHeader {
    parent_hash:        Hash,          // Hash of the previous block header
    block_number:       BlockNumber,   // Height of this block
    state_root:         Hash,          // Merkle root of the state trie after applying this block
    extrinsics_root:    Hash,          // Merkle root of the ordered extrinsic list
    timestamp:          Timestamp,     // Block production time
    validator:          PublicKey,     // Block proposer's public key
    signature:          Signature,     // Proposer's signature over the header hash
}
```

**Header Hash:** `SHA-256(SCALE(BlockHeader with signature = [0; 64]))`

The signature covers the hash of the header with the signature field zeroed.

### Block

```
Block {
    header:      BlockHeader,
    extrinsics:  Vec<Extrinsic>,    // Ordered list of extrinsics included in this block
}
```

### Extrinsic

An **Extrinsic** is a signed external action submitted to the chain. It encompasses both transactions (user-initiated) and inherents (validator-provided data like timestamps).

```
Extrinsic {
    version:    u8,              // Encoding version; MUST be 1 for this specification
    origin:     Origin,
    call:       Call,
    signature:  Option<ExtrinsicSignature>,
}

Origin {
    account_id: AccountId,
}

ExtrinsicSignature {
    signer:    PublicKey,
    signature: Signature,
    nonce:     u64,             // Account nonce; prevents replay attacks
    tip:       Balance,         // Optional priority fee
}
```

**Extrinsic Hash:** `SHA-256(SCALE(Extrinsic))`

### Call

A `Call` identifies the pallet and function to invoke, plus its encoded arguments.

```
Call {
    pallet_index:   u8,
    call_index:     u8,
    arguments:      Bytes,    // SCALE-encoded call arguments
}
```

---

## DDC Domain Types

These types represent the application-layer data specific to the diamond and commodity data use case.

### DDC Record

A **DDC Record** is the primary data unit stored on-chain. It represents a verifiable data entry for a physical or digital asset.

```
DdcRecord {
    record_id:      Hash,           // SHA-256 of (owner_id || asset_id || created_at)
    asset_id:       Bytes,          // External identifier (e.g., GIA certificate number)
    asset_type:     AssetType,
    owner:          AccountId,
    data_hash:      Hash,           // SHA-256 of the off-chain data payload
    data_uri:       Option<Bytes>,  // URI pointing to the off-chain payload (max 512 bytes)
    metadata:       Metadata,
    created_at:     Timestamp,
    updated_at:     Timestamp,
    status:         RecordStatus,
    attestations:   Vec<Attestation>,
}
```

### AssetType

```
AssetType {
    Diamond      = 0,
    ColuredStone = 1,
    Pearl        = 2,
    Metal        = 3,
    Other        = 255,
}
```

### RecordStatus

```
RecordStatus {
    Active      = 0,   // Record is live and current
    Transferred = 1,   // Ownership transferred; record superseded by successor
    Revoked     = 2,   // Revoked by issuing authority
    Archived    = 3,   // Retained for historical reference only
}
```

### Metadata

```
Metadata {
    entries: Vec<MetadataEntry>,   // Max 64 entries per record
}

MetadataEntry {
    key:   Bytes,   // UTF-8 string, max 128 bytes
    value: Bytes,   // max 1024 bytes
}
```

Implementations MUST reject records with duplicate metadata keys.

### Attestation

An **Attestation** is a third-party endorsement of a DDC Record issued by a registered Attestor.

```
Attestation {
    attestor:       AccountId,
    record_id:      Hash,
    attestation_id: Hash,          // SHA-256 of (attestor || record_id || issued_at)
    schema_id:      Hash,          // Identifies the attestation schema used
    payload_hash:   Hash,          // Hash of the attestation payload
    payload_uri:    Option<Bytes>, // URI to the full attestation payload
    issued_at:      Timestamp,
    expires_at:     Option<Timestamp>,
    signature:      Signature,
}
```

The attestation signature covers:
```
message = SHA-256(SCALE(Attestation with signature = [0; 64]))
```

---

## State Trie

DDC uses a **Merkle Patricia Trie (MPT)** for the state. The trie root is committed in every block header as `state_root`.

Keys in the trie are constructed as:

```
key = Blake2b-256(pallet_prefix || storage_item_prefix || item_key)
```

Where prefixes are defined per pallet. This matches the Substrate storage key derivation scheme.

The state trie root enables light clients to verify any piece of state using an inclusion proof of `O(log n)` nodes.

---

## Merkle Proof Format

```
MerkleProof {
    root:   Hash,
    leaf:   Bytes,              // The proven value
    path:   Vec<ProofNode>,
}

ProofNode {
    position: NodePosition,     // Left | Right
    hash:     Hash,
}

NodePosition {
    Left  = 0,
    Right = 1,
}
```

A proof is valid if applying the path of sibling hashes to the leaf hash produces the claimed root.

---

## Genesis Block

The genesis block MUST have:

- `block_number = 0`
- `parent_hash = Hash([0u8; 32])`  (all zeros)
- `extrinsics = []`
- `state_root` = hash of the initial state as defined by the genesis configuration

The genesis configuration is distributed as a JSON file alongside each network release. The `state_root` in the genesis header MUST match the deterministic trie root computed from the genesis configuration.

---

## Limits

| Parameter | Value |
|-----------|-------|
| Maximum block size | 5 MiB |
| Maximum extrinsics per block | 4096 |
| Maximum extrinsic size | 256 KiB |
| Maximum metadata entries per record | 64 |
| Maximum metadata key length | 128 bytes |
| Maximum metadata value length | 1024 bytes |
| Maximum data URI length | 512 bytes |
| Maximum attestations per record | 32 |

---

## Test Vectors

### BlockHeader Hash

Given:
```json
{
  "parent_hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "block_number": 1,
  "state_root":   "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
  "extrinsics_root": "0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
  "timestamp": 1722830400000,
  "validator": "0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d",
  "signature": "0x0000...0000"
}
```

Expected header hash (with zeroed signature): computed via `SHA-256(SCALE_encode(header))`.

Reference implementations MUST reproduce the same hash for this input. Concrete hash values are provided in the conformance test suite (`tests/conformance/`).

---

## References

- [SCALE Codec](https://docs.substrate.io/reference/scale-codec/)
- [Merkle Patricia Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)
- [Ed25519](https://www.rfc-editor.org/rfc/rfc8032)
- [BLS12-381](https://electriccoin.co/blog/new-snark-curve/)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
