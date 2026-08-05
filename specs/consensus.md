# Consensus Protocol

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

## Abstract

This document specifies the DDC-BFT consensus protocol: a Byzantine Fault Tolerant (BFT) consensus mechanism for ordering and finalizing blocks on the Diamond Data Chain. DDC-BFT is based on a round-based PBFT-style protocol adapted for a validator-set model with stake-weighted voting and deterministic finality.

---

## Definitions

| Term | Definition |
|------|------------|
| **Validator** | A node that has bonded sufficient stake and is part of the active validator set |
| **Validator Set** | The ordered set of validators eligible to participate in consensus for a given epoch |
| **Epoch** | A fixed window of blocks during which the validator set is stable; currently 3600 blocks (~1 hour at 1-second block time) |
| **Round** | A single attempt to finalize a block at a given height; multiple rounds may occur if the primary fails |
| **Primary** | The designated block proposer for a given round, selected deterministically from the validator set |
| **Replica** | Every validator that is not the primary in the current round |
| **Quorum** | A set of validators representing more than two-thirds of total bonded stake |
| **View** | A (height, round) pair uniquely identifying a consensus instance |
| **Equivocation** | Signing two conflicting messages for the same view |

---

## Validator Set

### Eligibility

An account becomes an eligible validator by:

1. Bonding at least **10,000 DDC** to the staking pallet.
2. Registering a session key (Ed25519 for consensus, BLS12-381 for aggregate signatures).
3. Being elected into the active set at the start of an epoch.

### Active Set Selection

At the end of each epoch, the top `N` candidates by bonded stake are elected as the active validator set. `N` is a governance parameter with a current value of **100**.

### Validator Set Changes

Validator set changes take effect at epoch boundaries. The new validator set is committed in the last block of the epoch as part of the `state_root`.

---

## Block Production

### Primary Selection

The primary for block at height `h` and round `r` is selected as:

```
primary_index = (h + r) mod |validator_set|
primary = validator_set[primary_index]
```

Where `validator_set` is the ordered active set for the current epoch, sorted by `AccountId` ascending.

### Block Proposal

1. The primary collects pending extrinsics from the mempool.
2. The primary constructs a `Block` according to the [Data Model](data-model.md).
3. The primary broadcasts a `Proposal` message to all replicas.

```
Proposal {
    view:      View,
    block:     Block,
    signature: Signature,    // Primary signs the block hash
}

View {
    height: BlockNumber,
    round:  u32,
}
```

---

## Consensus Phases

DDC-BFT proceeds in three phases per round: **Prepare**, **Commit**, and **Finalize**.

```
Primary                 Replicas
   │                       │
   │──── Proposal ─────────►│
   │                       │ (validate block)
   │◄─── PrepareVote ───────│
   │                       │
   │ (collect quorum)       │
   │──── PrepareQC ────────►│
   │                       │ (lock on block)
   │◄─── CommitVote ────────│
   │                       │
   │ (collect quorum)       │
   │──── CommitQC ─────────►│
   │                       │ (finalize block)
```

### Phase 1: Prepare

1. Each replica receives the `Proposal`.
2. The replica validates the block:
   - The block extends the current best chain.
   - The block number equals `current_height + 1`.
   - The timestamp is within ±5 seconds of the replica's local clock.
   - All extrinsics are valid per the state machine.
   - The primary signature is valid.
3. If valid, the replica broadcasts a `PrepareVote`.

```
PrepareVote {
    view:       View,
    block_hash: Hash,
    validator:  PublicKey,
    signature:  Signature,
}
```

4. The primary (or any node) aggregates `PrepareVote` messages into a **Prepare Quorum Certificate (PrepareQC)** once a quorum (>2/3 stake) is reached.

```
PrepareQC {
    view:            View,
    block_hash:      Hash,
    aggregate_sig:   AggregateSignature,   // BLS aggregate over PrepareVotes
    signers:         BitVec,               // Bitmap of participating validators
}
```

### Phase 2: Commit

1. Nodes that receive a valid `PrepareQC` broadcast a `CommitVote`.

```
CommitVote {
    view:       View,
    block_hash: Hash,
    validator:  PublicKey,
    signature:  Signature,
}
```

2. Once a quorum of `CommitVote` messages is collected, a **Commit Quorum Certificate (CommitQC)** is formed.

```
CommitQC {
    view:            View,
    block_hash:      Hash,
    aggregate_sig:   AggregateSignature,
    signers:         BitVec,
}
```

### Phase 3: Finalize

1. Any node that receives a valid `CommitQC` MUST finalize the block at the given height.
2. Finalized blocks are irreversible; the state machine updates are applied permanently.
3. The `CommitQC` is included in the next block header as `justification`.

---

## View Change (Round Change)

If no block is finalized within the **round timeout**, validators trigger a view change to the next round.

### Round Timeout

```
timeout(round) = base_timeout × 2^min(round, max_exponent)
```

Where `base_timeout = 1000ms` and `max_exponent = 6` (maximum timeout ~64 seconds).

### View Change Message

```
ViewChange {
    new_view:       View,
    last_commit_qc: Option<CommitQC>,  // Highest CommitQC the sender is aware of
    validator:      PublicKey,
    signature:      Signature,
}
```

### New View

When the new primary collects `ViewChange` messages from a quorum of validators, it constructs a `NewView` message and broadcasts it with a new proposal.

```
NewView {
    view:          View,
    view_changes:  Vec<ViewChange>,
    proposal:      Proposal,
}
```

The new proposal MUST extend the block with the highest known `PrepareQC`, or extend the current tip if no `PrepareQC` is known.

---

## Slashing

### Equivocation

A validator that signs two conflicting `PrepareVote` or `CommitVote` messages for the same `View` is slashed **5% of bonded stake**. Evidence MUST be submitted on-chain within 28 days.

```
EquivocationReport {
    offender:    AccountId,
    vote_1:      SignedVote,
    vote_2:      SignedVote,
}

SignedVote {
    view:       View,
    block_hash: Hash,
    phase:      VotePhase,     // Prepare | Commit
    signature:  Signature,
}
```

### Persistent Unavailability

A validator that participates in fewer than **10%** of rounds during an epoch is automatically removed from the validator set at epoch end and has **1% of bonded stake** slashed.

---

## Safety and Liveness

**Safety:** Two conflicting blocks can never both be finalized, because finalization requires a quorum CommitQC, and two quorums must share at least one honest validator that cannot commit to two conflicting blocks.

**Liveness:** Provided at least two-thirds of stake is controlled by online, honest validators, the protocol will always produce a new finalized block within `O(1)` rounds.

---

## Constants

| Parameter | Value | Description |
|-----------|-------|-------------|
| `EPOCH_LENGTH` | 3600 | Blocks per epoch |
| `MAX_VALIDATORS` | 100 | Maximum active validator set size |
| `MIN_BOND` | 10,000 DDC | Minimum stake to be a validator candidate |
| `BASE_TIMEOUT` | 1000 ms | Initial round timeout |
| `MAX_TIMEOUT_EXPONENT` | 6 | Maximum timeout doubling exponent |
| `EQUIVOCATION_SLASH` | 5% | Slash fraction for equivocation |
| `UNAVAILABILITY_SLASH` | 1% | Slash fraction for persistent unavailability |
| `UNAVAILABILITY_THRESHOLD` | 10% | Minimum round participation to avoid slash |
| `EVIDENCE_WINDOW` | 28 days | Maximum age of slashing evidence |

---

## References

- [Architecture Overview](architecture.md)
- [Data Model](data-model.md)
- [Governance](governance.md)
- [Practical Byzantine Fault Tolerance (Castro & Liskov, 1999)](https://pmg.csail.mit.edu/papers/osdi99.pdf)
- [HotStuff: BFT Consensus with Linearity and Responsiveness](https://arxiv.org/abs/1803.05069)
- [BLS Signatures](https://www.ietf.org/archive/id/draft-irtf-cfrg-bls-signature-05.txt)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
