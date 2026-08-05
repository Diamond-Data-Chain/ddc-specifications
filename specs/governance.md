# Governance

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

## Abstract

This document defines the on-chain governance methodology for the Diamond Data Chain (DDC) ecosystem. It covers proposal types, voting mechanics, quorum requirements, the upgrade process, and the roles of participants.

---

## Participants

### Token Holders

Any account holding DDC tokens may participate in governance by submitting and voting on proposals. Voting power is proportional to the voter's **bonded** (staked) DDC balance at the time of the snapshot.

### Council

The **Council** is an elected body of **13 members** that:

- Can fast-track emergency proposals.
- Has veto power over proposals that would harm network security.
- Manages the on-chain Treasury.

Council seats are filled by election every **6 months** (182 days). Any account with at least 1000 DDC bonded may stand for election.

### Technical Committee

The **Technical Committee** consists of the core development teams recognized by the Council. It has 7 seats. The Technical Committee:

- May propose emergency bug-fix upgrades.
- Reviews technical aspects of protocol upgrade proposals.

---

## Proposal Types

| Type | Description | Proposer | Threshold |
|------|-------------|----------|-----------|
| `StandardMotion` | General parameter changes, text motions | Any token holder | Simple majority (>50%) |
| `ProtocolUpgrade` | Runtime or consensus upgrade (major/minor) | Any token holder | Super-majority (>66.7%) |
| `EmergencyUpgrade` | Critical security fix; fast-tracked | Technical Committee | Council approval (>75% of council) |
| `TreasurySpend` | Spend from the on-chain Treasury | Any token holder | Simple majority |
| `CouncilElection` | Initiate a new council election | Any token holder | Simple majority |
| `TextProposal` | Record decisions or statements on-chain | Any token holder | Simple majority |

---

## Proposal Lifecycle

```
Submitted ──► Voting ──► Passed ──► Enacted
                │
                └──► Failed ──► (archived)
```

### 1. Submission

To submit a proposal, an account MUST lock a **proposal deposit** of **100 DDC**. The deposit:

- Is returned if the proposal passes.
- Is slashed if the proposal fails to reach quorum (see §5).
- Is returned (not slashed) if the proposal fails due to insufficient votes.

A proposal includes:

```
Proposal {
    proposal_id:   Hash,               // SHA-256(proposer || call_hash || block_number)
    proposer:      AccountId,
    proposal_type: ProposalType,
    call_hash:     Hash,               // Hash of the on-chain call to execute if passed
    call:          Option<Call>,       // Preimage (must be submitted before enactment)
    description:   Bytes,              // UTF-8, max 10240 bytes
    submitted_at:  BlockNumber,
    voting_starts: BlockNumber,        // = submitted_at + SUBMISSION_DELAY
}
```

`SUBMISSION_DELAY = 7200 blocks` (~2 hours at 1-second block time). This allows the community to review the proposal before voting begins.

### 2. Voting

The voting period for standard proposals is **28 days** (2,419,200 blocks). During this period, token holders cast votes.

```
Vote {
    proposal_id: Hash,
    voter:       AccountId,
    vote:        VoteDirection,
    conviction:  Conviction,
}

VoteDirection {
    Aye  = 0,
    Nay  = 1,
    Abstain = 2,
}
```

#### Conviction Voting

Token holders may multiply their voting power by locking their tokens for an extended period after the vote:

| Conviction | Vote Multiplier | Lock Period |
|------------|----------------|-------------|
| `None` | 0.1× | No lock |
| `Locked1x` | 1× | 8 days |
| `Locked2x` | 2× | 16 days |
| `Locked3x` | 3× | 32 days |
| `Locked4x` | 4× | 64 days |
| `Locked5x` | 5× | 128 days |
| `Locked6x` | 6× | 256 days |

Abstain votes contribute to quorum but do not influence the Aye/Nay result.

### 3. Tallying

At the end of the voting period:

```
aye_power  = Σ (voter_bonded_balance × conviction_multiplier) for Aye votes
nay_power  = Σ (voter_bonded_balance × conviction_multiplier) for Nay votes
total_issuance = total bonded supply at snapshot block
turnout = (aye_power + nay_power + abstain_power) / total_issuance
```

#### Adaptive Quorum Biasing

DDC uses an adaptive quorum mechanism that adjusts the passing threshold based on turnout:

| Turnout | Required Aye share (StandardMotion) |
|---------|-------------------------------------|
| < 25% | 75% |
| 25–50% | Linear interpolation between 75% and 50% |
| ≥ 50% | 50% (simple majority) |

For `ProtocolUpgrade` proposals, the required Aye share is fixed at **66.7%** regardless of turnout, with a minimum quorum of **25%** of total bonded supply.

### 4. Enactment

If a proposal passes, there is a mandatory **enactment delay** before the call is executed:

| Proposal Type | Enactment Delay |
|---------------|----------------|
| `StandardMotion` | 2 days |
| `ProtocolUpgrade` | 14 days |
| `EmergencyUpgrade` | 0 blocks (immediate) |
| `TreasurySpend` | 2 days |

During the enactment delay, the call preimage MUST be submitted on-chain if it has not been already. Failure to submit the preimage before enactment causes the proposal to expire without effect.

---

## Council Operations

### Council Voting

Council members vote on motions submitted to the Council. A motion passes when:

- For standard motions: a simple majority (>50%) of council members vote Aye.
- For fast-track / emergency motions: >75% of council members vote Aye.

Council votes have no conviction multiplier.

### Council Veto

The Council MAY veto any `StandardMotion` or `ProtocolUpgrade` proposal within the enactment delay period. A veto requires >75% Council approval. Vetoed proposals are archived and the proposal deposit is returned.

The Technical Committee MAY override a veto for `EmergencyUpgrade` proposals with a unanimous vote.

---

## Quorum Failure and Deposit Slashing

If a proposal enters a vote and turnout does not reach the minimum quorum threshold (**5% of total bonded supply**) by the end of the voting period, the proposal FAILS and the proposer's deposit is slashed and sent to the Treasury.

If the proposal fails due to insufficient Aye votes (quorum met but Nay wins), the deposit is returned.

---

## Treasury

The on-chain Treasury is funded by:

- A portion of transaction fees (**20%** of all fees).
- Slashed validator and governance deposits.

Treasury funds are released via `TreasurySpend` proposals. The Treasury MAY fund ecosystem development, audits, marketing, and other public goods.

---

## Protocol Upgrade Process

### Standard Upgrade Path

1. A GitHub Discussion or RFC is opened (off-chain deliberation).
2. A `ProtocolUpgrade` proposal is submitted with the new runtime/consensus call.
3. The 28-day voting period concludes with a super-majority Aye.
4. A 14-day enactment delay allows node operators to upgrade their software.
5. The upgrade activates at the specified block height.

### Emergency Upgrade Path

1. The Technical Committee identifies a critical security vulnerability.
2. The Technical Committee submits an `EmergencyUpgrade` proposal.
3. The Council votes; >75% Aye is required.
4. Immediate enactment once the Council vote passes.
5. A post-mortem MUST be published within 7 days.

---

## Governance Parameters

All governance parameters are themselves subject to change via `StandardMotion` proposals.

| Parameter | Value |
|-----------|-------|
| `SUBMISSION_DELAY` | 7200 blocks (~2 hours) |
| `VOTING_PERIOD` | 2,419,200 blocks (~28 days) |
| `STANDARD_ENACTMENT_DELAY` | 172,800 blocks (~2 days) |
| `UPGRADE_ENACTMENT_DELAY` | 1,209,600 blocks (~14 days) |
| `PROPOSAL_DEPOSIT` | 100 DDC |
| `MIN_QUORUM` | 5% of total bonded supply |
| `COUNCIL_SIZE` | 13 members |
| `TECHNICAL_COMMITTEE_SIZE` | 7 members |
| `COUNCIL_TERM` | 182 days |
| `TREASURY_FEE_SHARE` | 20% |

---

## References

- [Architecture Overview](architecture.md)
- [Consensus Protocol](consensus.md)
- [Glossary](glossary.md)
- [Polkadot Governance](https://wiki.polkadot.network/docs/learn-governance) (inspiration)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
