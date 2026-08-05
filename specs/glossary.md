# Glossary

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

This glossary defines terms used across the DDC specification set. Where a term has a precise technical meaning within DDC, that definition supersedes common usage.

---

## A

**Account** – A cryptographic identity on the DDC network, identified by an `AccountId` (32-byte hash of a public key). Accounts hold DDC token balances and can send extrinsics.

**AccountId** – A 32-byte identifier derived as `Blake2b-256(public_key)`.

**Aggregate Signature** – A BLS12-381 signature that compactly proves that multiple validators have signed a common message. Used in Quorum Certificates.

**Archive Node** – A full node that retains all historical state without pruning. Required for answering historical queries.

**Attestation** – A third-party endorsement of a DDC Record, issued and cryptographically signed by a registered Attestor.

**Attestor** – An entity (account) registered on-chain that is authorized to issue Attestations. Attestors may represent certification laboratories, customs authorities, or other trusted parties.

---

## B

**Balance** – A `u128` value representing a DDC token amount in the smallest unit (pico-DDC, 10⁻¹²). 1 DDC = 1,000,000,000,000 pico-DDC.

**Block** – The fundamental unit of the DDC chain, consisting of a `BlockHeader` and an ordered list of `Extrinsic`s.

**Block Number** – A monotonically increasing `u64` counter starting at 0 for the genesis block.

**Block Store** – The persistent, append-only database on a full node that stores all blocks and associated state.

**BLS12-381** – An elliptic curve pairing used for aggregate signatures in DDC-BFT consensus.

**Bonded Balance** – DDC tokens locked in the staking pallet as a validator or delegator bond.

**Bootstrap Node** – A well-known, stable full node used by new peers to join the network. Not a privileged role in the protocol.

**Byzantine Fault Tolerant (BFT)** – A property of a distributed system that can tolerate a fraction of participants acting arbitrarily maliciously.

---

## C

**Call** – An encoded instruction to invoke a specific function in a specific pallet, packaged inside an `Extrinsic`.

**CommitQC** – A Commit Quorum Certificate: a BLS aggregate signature from a quorum of validators over a `CommitVote`, used to finalize a block.

**CommitVote** – A signed message from a validator indicating that it has observed a `PrepareQC` and is committing to a block.

**Consensus** – The process by which validators agree on the ordering and content of blocks. DDC uses **DDC-BFT**.

**Controller Account** – An account with permission to manage staking operations (bond, unbond, nominate) on behalf of a Stash Account.

**Conviction Voting** – A governance mechanism allowing token holders to amplify their vote power by locking tokens for an extended period.

**Council** – An elected body of 13 members with special governance powers, including fast-tracking emergency proposals and managing the Treasury.

---

## D

**DDC** – Diamond Data Chain. The protocol and network defined by these specifications.

**DDC-BFT** – The Byzantine Fault Tolerant consensus protocol used by DDC. A PBFT-style, round-based protocol with stake-weighted voting and deterministic finality.

**DDC Record** – The primary application-layer data unit: a verifiable, on-chain record representing a physical or digital asset.

**DHT** – Distributed Hash Table. DDC uses a Kademlia DHT for peer discovery.

**Ed25519** – An elliptic curve signature scheme used for node identity keys, session keys, and extrinsic signatures.

**Enactment Delay** – The mandatory waiting period after a governance proposal passes before the corresponding on-chain call is executed.

**Epoch** – A fixed period of 3,600 blocks (~1 hour) during which the active validator set is stable.

**Equivocation** – Signing two conflicting messages (e.g., two different `PrepareVote`s) for the same `View`. A slashable offense.

**Extrinsic** – A signed or unsigned message submitted to the DDC chain from outside. Encompasses user transactions and inherent data.

---

## F

**Finality** – The property that a finalized block can never be reverted. DDC-BFT provides deterministic (instant) finality upon the accumulation of a `CommitQC`.

**Full Node** – A node that stores the complete chain state and validates all blocks and transactions.

---

## G

**Genesis Block** – The first block of the DDC chain (block number 0). Its state root is defined by the genesis configuration file.

**Governance** – The on-chain mechanism by which DDC token holders and the Council collectively make decisions about protocol changes and Treasury spending.

---

## H

**Hash** – A 32-byte SHA-256 digest. Used for block hashes, record IDs, and all other integrity-critical identifiers.

---

## I

**Inherent** – An unsigned extrinsic inserted by the block proposer, carrying data like the block timestamp.

**Initial Block Download (IBD)** – The process a new node uses to synchronize the full chain from peers before participating in consensus.

---

## J

**Justification** – The `CommitQC` included in a block header that proves the previous block was finalized by a quorum.

---

## K

**Kademlia** – A peer-to-peer DHT protocol used by DDC for node discovery and routing.

---

## L

**libp2p** – A modular networking framework used by DDC for peer-to-peer communication.

**Light Node** – A node that stores only block headers and verifies data using Merkle proofs, without storing the full chain.

---

## M

**Mempool** – The in-memory pool of pending (unconfirmed) extrinsics maintained by each full node.

**Merkle Patricia Trie (MPT)** – The data structure used to commit the DDC state. Enables efficient Merkle proofs for any state entry.

**Merkle Proof** – A cryptographic proof that a specific value is included in a Merkle trie at a particular root.

**Metadata** – Key-value pairs attached to a DDC Record providing additional descriptive or provenance information.

**Multiaddr** – A self-describing network address format used by libp2p.

---

## N

**Noise Protocol** – A framework for building cryptographic protocols. DDC uses the Noise XX handshake for encrypting peer-to-peer connections.

**Nonce** – A monotonically increasing counter associated with each account, included in extrinsics to prevent replay attacks.

---

## P

**Pallet** – A modular runtime component implementing a specific set of state and logic (analogous to a smart contract module). DDC runtime is composed of pallets.

**Peer ID** – A unique identifier for a node, derived from its Ed25519 public key. Represented as a Multihash-encoded string.

**Peer Scoring** – A mechanism by which nodes rate their peers' behavior and disconnect low-scoring peers.

**PrepareQC** – A Prepare Quorum Certificate: a BLS aggregate signature from a quorum of validators over a `PrepareVote`, indicating readiness to commit a block.

**PrepareVote** – A signed message from a validator indicating that it has validated a proposed block and is ready to commit.

**Primary** – The validator designated to propose a block in a given round of consensus.

**Proposal Deposit** – DDC tokens locked when submitting a governance proposal. Returned or slashed depending on the outcome.

---

## Q

**Quorum** – A set of validators whose aggregate stake exceeds two-thirds of total bonded stake. Required for `PrepareQC` and `CommitQC`.

**Quorum Certificate (QC)** – A BLS aggregate signature over votes from a quorum of validators. See `PrepareQC` and `CommitQC`.

---

## R

**Replica** – Any validator that is not the primary in a given consensus round.

**Round** – A single attempt to finalize a block at a given height. Multiple rounds may occur if the primary fails or the block is invalid.

**Round Timeout** – The time a validator waits before triggering a view change to the next round. Doubles with each consecutive failure (up to a cap).

---

## S

**SCALE** – Simple Concatenated Aggregate Little-Endian codec. The binary encoding format for all on-chain and network data in DDC.

**Session Key** – A hot key used by validators to sign consensus messages. MUST be rotated periodically.

**SHA-256** – The cryptographic hash function used for all DDC hash computations.

**Slashing** – The forcible reduction of a validator's bonded stake as a penalty for protocol violations (equivocation, persistent unavailability).

**Stash Account** – The account that holds a validator's bonded funds, kept in cold storage. Managed via a Controller Account.

**State Root** – The Merkle Patricia Trie root committed in a block header, representing the entire DDC state after applying that block.

---

## T

**Technical Committee** – A 7-member body of core developers with special powers to propose and fast-track emergency upgrades.

**Timestamp** – A `u64` value representing a Unix timestamp in milliseconds (UTC).

**Transaction Pool** – See *Mempool*.

**Treasury** – The on-chain fund used for ecosystem development, audits, and other public goods, managed via governance.

---

## V

**Validator** – A node that has bonded sufficient stake and is part of the active validator set; participates in DDC-BFT consensus.

**Validator Set** – The ordered set of validators active in the current epoch, selected by stake at epoch boundaries.

**View** – A `(height, round)` pair uniquely identifying a consensus instance.

**View Change** – The process by which validators move to a new round when the current round's primary fails to produce a valid block.

---

## W

**WebSocket** – A persistent, bidirectional communication protocol supported by the DDC Node API for real-time event subscriptions.

---

## References

- [Architecture Overview](architecture.md)
- [Data Model](data-model.md)
- [Consensus Protocol](consensus.md)
- [Network Protocol](network-protocol.md)
- [Node API](node-api.md)
- [Node Operations](node-operations.md)
- [Governance](governance.md)
