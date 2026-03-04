    ZIP: Unassigned
    Title: Zcash Shielded Coinholder Voting
    Owners: Dev Ojha <dojha@berkeley.edu>
            Roman Akhtariev <ackhtariev@gmail.com>
            Adam Tucker <adamleetucker@outlook.com>
            Greg Nagy <greg@dhamma.works>
    Status: Draft
    Category: Process
    Created: 2026-03-04
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/???>

# Terminology

The key words "MUST", "REQUIRED", "MUST NOT", "SHOULD", and "MAY" in this
document are to be interpreted as described in BCP 14 [^BCP14] when, and
only when, they appear in all capitals.

The terms below are to be interpreted as follows:

Vote chain
: A purpose-built Cosmos SDK blockchain that serves as the single source
of truth for all voting operations. The vote chain stores vote
commitments, nullifier sets, and encrypted share accumulators, and
verifies zero-knowledge proofs for each transaction type.

Voting round
: A complete instance of a coinholder vote, from round creation through
tally. Each round is scoped to a single Zcash mainnet snapshot and a
fresh Election Authority key.

Vote round ID
: A unique identifier for a voting round, derived from the round's
  snapshot and proposal parameters. See [Round Creation] for the
  computation.

Poll runner
: The entity responsible for conducting a voting round: selecting the
snapshot height, coordinating validators, publishing the round, and
overseeing the tally.

Vote manager
: The on-chain role that creates voting rounds. Only the vote manager
can publish new rounds via `MsgCreateVotingSession`.

Bootstrap operator
: The entity that generates the vote chain genesis, funds validators
from the token reserve, and controls initial consensus power
distribution.

Validator
: A vote chain consensus participant that runs the chain, participates
in the EA key ceremony, and contributes to the automatic tally. Each
validator maintains three keypairs: consensus, account, and Pallas.

Voter
: Any holder of one or more Orchard notes at the snapshot height. Voters
interact with the system via a wallet application.

Vote Authority Note (VAN)
: A note created on the vote chain during delegation (Phase 1) that
represents a voter's delegated voting weight. A VAN is consumed during
voting (Phase 2) and a new VAN is produced with the relevant proposal
bit cleared.

Vote Commitment (VC)
: A commitment created during Phase 2 that binds the voter's choice and
encrypted shares for a given proposal. Vote commitments are stored in
the Vote Commitment Tree.

Vote Commitment Tree (VCT)
: A Poseidon Merkle tree on the vote chain that stores all vote
commitments for a given round.

Election Authority (EA)
: A logical role whose secret key $\mathsf{ea\_sk}$ is used for El Gamal
  encryption [^elgamal] of vote shares. In the current design, all ACK'd
  validators hold $\mathsf{ea\_sk}$ for the round.

EA key ceremony
: A per-round protocol that produces a fresh El Gamal keypair
$(\mathsf{ea\_sk}, \mathsf{ea\_pk})$ distributed to eligible validators.
See [EA Key Ceremony].

Snapshot height
: The Zcash mainnet block height at which eligible Orchard note balances
  are captured. See [Snapshot Configuration] for constraints.

Helper server
: A service embedded in validator nodes that receives encrypted vote
shares from wallet clients, buffers them, and submits them to the vote
chain with randomized delays to provide temporal unlinkability.

For definitions of cryptographic terms including _alternate nullifier_,
_nullifier non-membership tree_, _nullifier domain_, _pool snapshot_, and
_claim_, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
PIR-related terms, see the PIR for Nullifier Exclusion Proofs ZIP [^draft-pir].

# Abstract

This ZIP specifies the end-to-end process for conducting a
privacy-preserving Zcash coinholder vote. It covers the vote chain
infrastructure, poll setup, the voting process from delegation through
tally, and result verification. The system uses a purpose-built Cosmos SDK
chain as the single source of truth, with Zcash mainnet snapshots providing
eligible balances. Votes are cast using zero-knowledge proofs and El Gamal
encryption so that voter identity, individual vote amounts, and total
holdings remain hidden, while anyone can verify the tally via DLEQ
proofs [^cp92].

# Motivation

ZIP 1016 [^zip-1016] establishes a Coinholder-Controlled Fund funded by 12%
of block rewards, requiring coinholder votes to approve grant proposals.
Conducting such votes requires a concrete process specification that a new
poll runner can follow end-to-end without needing to understand the
underlying cryptographic circuits.

This ZIP separates operational concerns — infrastructure setup, role
assignment, round lifecycle, timing — from the cryptographic protocol
details specified in the companion ZIPs. A poll runner reads this document
and knows what to do; a circuit implementer reads the companion ZIPs and
knows what to prove.

# Requirements

- A new poll runner can conduct a vote by following this specification and
  the referenced companion ZIPs.
- **Privacy**: voter identity, individual vote amounts, and total holdings
  remain hidden from all parties including validators.
- **Verifiability**: anyone can audit the tally by verifying the DLEQ
  proofs without trusting the Election Authority or validators.
- **Liveness**: voting proceeds with partial validator availability (at
  least one-third of validators ACK the ceremony).
- **Accessibility**: voters can participate in a single online session —
  come online once, delegate and vote, then go offline.
- **Minimum voting weight**: one ballot corresponds to 12,500,000 zatoshi
  (0.125 ZEC). Notes below this threshold are not eligible.

# Non-requirements

The following are explicitly out of scope for this ZIP:

- Governance policy decisions such as proposal eligibility, quorum
  requirements, and fund disbursement rules (see ZIP 1016 [^zip-1016]).
- The cryptographic proof-of-balance protocol: note ownership proofs,
  alternate nullifier derivation, and nullifier non-membership tree
  construction (see [^draft-balance-proof]).
- Vote proof and reveal proof circuits, delegation protocol, share
  splitting, and El Gamal encryption scheme (see [^draft-voting-protocol]).
- PIR protocol details: YPIR+SP construction, three-tier data structure,
  and query mechanics (see [^draft-pir]).
- On-chain accountable voting (see [^draft-onchain-voting]).

# Specification

## System Overview

The coinholder voting system operates on a purpose-built Cosmos SDK vote
chain that serves as the single source of truth for all voting operations.
Zcash mainnet snapshots provide the set of eligible Orchard note balances.

The vote chain stores:

- A **Vote Commitment Tree** (VCT): a Poseidon Merkle tree of vote
  commitments.
- Three **nullifier sets**: governance nullifiers (alternate nullifiers
  from note claims), VAN nullifiers (from delegation consumption), and
  share nullifiers (from share reveals).
- An **encrypted share accumulator** per (proposal, decision): the
  homomorphic sum of El Gamal ciphertexts for each vote option.

The vote chain verifies a zero-knowledge proof for each transaction type:
delegation, vote, and share reveal. The proof circuits are specified in the
Voting Protocol ZIP [^draft-voting-protocol].

Voting proceeds in five phases, detailed in [Voting Window]:

1. **Delegation** — prove note ownership, delegate to hotkey.
2. **Voting** — cast encrypted vote using hotkey.
3. _(reserved)_
4. **Share submission** — wallet sends encrypted shares to helper servers.
5. **Share reveal** — helper servers submit shares to chain with proof.

## Roles and Authorities

### Bootstrap Operator

The bootstrap operator generates the vote chain genesis block, funds
validators from the token reserve, and controls initial consensus power
distribution. Funding amount determines each validator's consensus voting
power.

### Vote Manager

The vote manager creates voting rounds by submitting
`MsgCreateVotingSession` transactions. Only the vote manager role can
publish new rounds.

Assignment rules:

- **Bootstrap phase**: any bonded validator MAY claim the vote manager
  role.
- **Subsequent rounds**: the current vote manager retains the role, or any
  bonded validator MAY reassign it.

### Validator

Validators participate in consensus, the EA key ceremony, and automatic
tally computation. Each validator maintains three keypairs:

- **Consensus keypair**: used for CometBFT consensus.
- **Account keypair**: used for submitting chain transactions.
- **Pallas keypair**: used for ECIES key exchange during the EA ceremony.

Validators join the network via the automated `join.sh` script or by
building from source. See [Onboarding Validators].

### Voter

Any holder of Orchard notes at the snapshot height. Voters interact with
the voting system through a wallet application. The wallet handles proof
generation, delegation, and share submission.

### Helper Server

Helper servers are embedded in validator nodes. They receive encrypted vote
shares from wallet clients and submit them to the vote chain with
randomized delays, providing temporal unlinkability between wallet
submissions and on-chain transactions.

### Authority Summary

| Action                     | Bootstrap Op. | Vote Manager | Validator | Voter |
| -------------------------- | :-----------: | :----------: | :-------: | :---: |
| Generate genesis           |       X       |              |           |       |
| Fund validators            |       X       |              |           |       |
| Create voting round        |               |      X       |           |       |
| Participate in consensus   |               |              |     X     |       |
| EA key ceremony            |               |              |     X     |       |
| Compute tally              |               |              |     X     |       |
| Delegate voting weight     |               |              |           |   X   |
| Cast vote                  |               |              |           |   X   |
| Submit shares (via helper) |               |              |     X     |       |
| Verify tally               |       X       |      X       |     X     |   X   |

## Vote Chain Infrastructure

### Genesis Validator Setup

Prerequisites:

- Linux or macOS
- Go 1.24 or later
- Rust 1.83 or later
- C toolchain (GCC or Clang)

Build the vote chain binary:

    git clone https://github.com/z-cale/zally
    cd zally/sdk
    make install-ffi

Initialize the chain:

    zallyd init <moniker> --chain-id zvote-1

This generates a Pallas keypair for the EA ceremony and configures the
REST API on port 1318.

Network ports:

| Port  | Protocol | Exposure | Purpose              |
| ----- | -------- | -------- | -------------------- |
| 26656 | P2P      | Public   | Peer-to-peer gossip  |
| 26657 | RPC      | Local    | CometBFT RPC         |
| 1318  | REST     | Public   | Application REST API |

Start the chain and verify that block production begins:

    zallyd start
    curl http://localhost:26657/status

### Onboarding Validators

**Automated flow** (`join.sh`):

1. Download prebuilt binaries.
2. Discover the network via service discovery (see [Service Discovery]).
3. Fetch genesis and sync to current height.
4. Generate consensus, account, and Pallas keypairs.
5. Request funding from the bootstrap operator via the admin UI.
6. Auto-register with `MsgCreateValidatorWithPallasKey`.

**Source-based flow**:

    mise run validator:join

**Funding**: the bootstrap operator sends tokens to the validator's account
address. The funding amount equals the validator's consensus voting power.

### Nullifier Service (PIR Server)

The nullifier service provides nullifier exclusion proofs to voters
via PIR.

**Bootstrap**:

1. Ingest Orchard nullifiers from Zcash mainnet via `lightwalletd`.
2. Build the Indexed Merkle Tree (nullifier non-membership tree) as
   specified in [^draft-balance-proof].
3. Prepare the three-tier PIR database as specified in [^draft-pir].

**Serve**: expose a query endpoint for voters to privately retrieve
exclusion proofs. See [^draft-pir] for the YPIR+SP protocol and query
mechanics.

### Service Discovery

An Edge Config registry (hosted on Vercel) stores:

- Validator P2P addresses and REST API URLs.
- PIR server URLs.

Wallet clients and the `join.sh` script discover the network through this
registry.

## Conducting a Voting Round

### Snapshot Configuration

The poll runner selects a Zcash mainnet snapshot height subject to these
constraints:

- The height MUST be at or after NU5 activation (Orchard is required).
- The height MUST be a multiple of 10.

Selecting the snapshot triggers:

1. The PIR server rebuilds its nullifier non-membership tree at that
   height.
2. The note commitment tree root ($\mathsf{nc\_root}$) and nullifier IMT
   root ($\mathsf{nullifier\_imt\_root}$) are captured at the snapshot
   height.

### Round Creation

The vote manager publishes a new round via `MsgCreateVotingSession` with
the following parameters:

- `snapshot_height`: the selected Zcash mainnet block height.
- `snapshot_blockhash`: the block hash at `snapshot_height`.
- `proposals`: 1 to 16 proposals, each with 2 to 8 labeled options (e.g.,
  "Support" / "Oppose").
- `vote_end_time`: deadline for all voting phases.
- `nullifier_imt_root`: root of the nullifier non-membership tree at
  snapshot.
- `nc_root`: Orchard note commitment tree root at snapshot.
- `verification_keys`: verification keys for the ZKP circuits (delegation,
  vote, share reveal).

The vote round ID is computed as:

$$
\mathsf{vote\_round\_id} = \mathsf{Blake2b}(\mathsf{snapshot\_height} \| \mathsf{snapshot\_blockhash} \| \mathsf{proposals\_hash} \| \mathsf{vote\_end\_time} \| \mathsf{nullifier\_imt\_root} \| \mathsf{nc\_root})
$$

where Blake2b [^blake2] is used with a 256-bit output.

The round enters the **PENDING** state, awaiting the EA key ceremony.

### EA Key Ceremony

Each voting round requires a fresh El Gamal keypair
$(\mathsf{ea\_sk}, \mathsf{ea\_pk})$ produced by an automated ceremony:

1. **Eligibility snapshot**: all validators with a registered Pallas public
   key at the time of round creation are eligible.

2. **Dealer selection**: the next block proposer is automatically selected
   as the dealer. The dealer generates $\mathsf{ea\_sk}$, computes
   $\mathsf{ea\_pk} = \mathsf{ea\_sk} \cdot G$, and encrypts
   $\mathsf{ea\_sk}$ to each eligible validator using ECIES [^ecies]
   (ephemeral ECDH on Pallas curve with ChaCha20-Poly1305 symmetric
   encryption).

3. **Auto-ACK**: each eligible validator decrypts $\mathsf{ea\_sk}$,
   verifies that $\mathsf{ea\_sk} \cdot G = \mathsf{ea\_pk}$, and submits
   an ACK message via `PrepareProposal`.

4. **Confirmation**:
   - _Fast path_: all eligible validators ACK — the ceremony confirms
     immediately.
   - _Timeout path_ (30 minutes): if at least one-third of eligible
     validators have ACK'd, the ceremony confirms and non-ACK'd validators
     are stripped from the round. Non-ACK'd validators increment a
     consecutive-miss counter; after 3 consecutive misses, the validator is
     jailed.
   - _Failure_: if fewer than one-third ACK within the timeout, the
     ceremony resets and a new dealer is selected.

5. On successful confirmation, the round transitions from **PENDING** to
   **ACTIVE** and voting can begin.

### Voting Window

The voting window spans five phases. This section describes the operational
flow; see the Voting Protocol ZIP [^draft-voting-protocol] for circuit
specifications and cryptographic details.

#### Phase 1 — Delegation

The voter proves ownership of one or more Orchard notes at the snapshot
height and delegates voting weight to a locally-generated hotkey. For each
note:

1. The voter retrieves a nullifier exclusion proof from the PIR server
   (see [^draft-pir]) to prove the note was unspent at the snapshot.
2. The voter generates a claim proof as specified in
   [^draft-balance-proof], demonstrating note ownership and revealing the
   note's governance alternate nullifier.
3. A delegation transaction is submitted to the vote chain, creating a
   Vote Authority Note (VAN) bound to the hotkey.

The vote chain records the governance alternate nullifier to prevent
double-delegation of the same note.

#### Phase 2 — Voting

Using the hotkey, the voter casts a vote on one or more proposals:

1. The hotkey consumes a VAN and produces a new VAN with the relevant
   proposal bit cleared, plus a Vote Commitment (VC) containing encrypted
   shares under $\mathsf{ea\_pk}$.
2. The vote transaction includes a zero-knowledge proof that the VC is
   well-formed and the encrypted shares are consistent with the voter's
   weight.
3. The VC is inserted into the Vote Commitment Tree.

The VAN nullifier is recorded to prevent double-voting on the same
proposal.

#### Phase 4 — Share Submission

After voting, the wallet sends individual encrypted shares to one or more
helper servers. Shares are encrypted under $\mathsf{ea\_pk}$ using El
Gamal.

#### Phase 5 — Share Reveal

Helper servers construct a proof that each share belongs to a valid VC in
the Vote Commitment Tree and submit the share to the vote chain at a
randomized delay. The vote chain:

1. Verifies the share reveal proof.
2. Records the share nullifier to prevent double-reveal.
3. Accumulates the El Gamal ciphertext homomorphically into the per
   (proposal, decision) aggregate.

### Tally and Results

After `vote_end_time` passes, the round transitions to the **TALLYING**
state:

1.  The block proposer loads $\mathsf{ea\_sk}$ and decrypts the aggregate
    ciphertext for each (proposal, decision) pair:

    $$
    \mathsf{total\_value} \cdot G = C_{2,\text{agg}} - \mathsf{ea\_sk} \cdot C_{1,\text{agg}}
    $$

2.  The proposer recovers $\mathsf{total\_value}$ from
    $\mathsf{total\_value} \cdot G$ using baby-step-giant-step discrete
    logarithm (feasible because $\mathsf{total\_value}$ is bounded by total
    ZEC supply).

3.  The proposer submits `MsgSubmitTally` with the recovered values and a
    Chaum-Pedersen DLEQ proof [^cp92] demonstrating correct decryption.

4.  Results are queryable via the REST API:

        GET /zally/v1/tally-results/{round_id}

    The response contains, for each (proposal, decision) pair, the
    `total_value` in zatoshi. Clients map each `vote_decision` index back
    to the option label defined in the proposal (e.g., "Support" /
    "Oppose").

## Timing and Deadlines

| Parameter                      | Value        | Notes                         |
| ------------------------------ | ------------ | ----------------------------- |
| Ceremony deal timeout          | ~30 blocks   | TBD; time for dealer message  |
| ACK phase timeout              | 30 minutes   | Fixed                         |
| Consecutive ceremony miss jail | 3 misses     | Validator jailed, not slashed |
| Voting window                  | Configurable | Set by `vote_end_time`        |
| Tally computation              | Automatic    | Triggered after window closes |
| Slashing fractions             | 0            | Jailing only, no token burns  |

## Verification and Auditing

Anyone MAY run a **monitoring node** — a full chain replica that does not
participate in consensus — to independently verify all aspects of a voting
round:

- Verify every zero-knowledge proof submitted in delegation, vote, and
  share reveal transactions.
- Track all three nullifier sets (governance, VAN, share) for
  double-spending.
- Recompute the aggregate El Gamal ciphertexts per (proposal, decision)
  from individual share reveals.
- Verify the DLEQ proof in `MsgSubmitTally` to confirm correct decryption.

The $\mathsf{ea\_sk}$ file for each round is retained indefinitely
(32 bytes per key) to allow future retally or audit. Keys are never
deleted.

No trust in the Election Authority or validators is required for tally
verification: the DLEQ proof is independently checkable by any party with
access to the chain state.

# Rationale

**Separate vote chain (not Zcash mainnet)**: the vote chain is purpose-built
for governance with ZKP-optimized state transitions (Poseidon hashing, custom
transaction types). Zcash mainnet's transaction throughput and scripting model
are not designed for interactive multi-phase voting protocols.

**Per-round EA key ceremony**: scoping $\mathsf{ea\_sk}$ to a single round
limits the impact of key compromise to that round only. Validator rotation
between rounds is handled naturally — departing validators cannot decrypt
future rounds. This avoids the complexity of re-initialization or long-lived
key management.

**Helper servers**: mobile wallets cannot reliably perform background
computation or maintain persistent connections. Helper servers buffer
encrypted shares and submit them with randomized delays, allowing voters to
complete all interaction in a single session.

**16 shares per vote**: balances privacy amplification against bandwidth
cost. This value is expected to increase in future revisions as bandwidth
constraints relax.

# Security Considerations

**Trust model**: all validators that ACK the EA ceremony hold
$\mathsf{ea\_sk}$ for the round. If any validator is compromised, vote
amount privacy is broken for that round (the adversary can decrypt
individual shares). However, voter identity remains protected because
alternate nullifiers are unlinkable to on-chain spending.

**Key isolation**: each round uses a different $\mathsf{ea\_sk}$. A
validator that departs or is compromised after one round cannot decrypt
votes in subsequent rounds.

**Spam resistance**: each delegation requires a valid zero-knowledge proof
(computationally expensive to generate) and the minimum voting weight of
0.125 ZEC (12,500,000 zatoshi) limits the number of ballots an adversary
can create.

**Trusted dealer**: the current ceremony uses a single dealer that
generates $\mathsf{ea\_sk}$. The dealer learns $\mathsf{ea\_sk}$ by
construction. A future upgrade path is threshold secret sharing (t-of-n
with Feldman commitments), followed by distributed key generation (DKG)
that eliminates the trusted dealer entirely.

**Helper server trust**: helper servers see encrypted shares and the
voter's decision index, but not plaintext amounts or voter identity. The
trust requirement is that helper servers do not leak timing metadata that
could link a wallet submission to its on-chain share reveal.

**Validator power distribution**: the bootstrap operator controls initial
power distribution via funding amounts. An even distribution across
validators reduces the risk of consensus capture.

**Post-quantum considerations**: El Gamal encryption is breakable by a
quantum adversary with a sufficiently large quantum computer. A successful
quantum attack would expose individual share amounts and delegation
amounts for the affected round. Hotkeys are never published on-chain and
remain safe. Post-quantum migration is out of scope for this ZIP.

# Reference implementation

[z-cale/zally](https://github.com/z-cale/zally) — a Go and Rust
implementation built on Cosmos SDK with Halo 2 zero-knowledge proof
circuits.

# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-voting-protocol.md)

[^draft-pir]: [Draft ZIP: Private Information Retrieval for Nullifier Exclusion Proofs](draft-valargroup-gov-pir.md)

[^draft-onchain-voting]: [Draft ZIP: On-chain Accountable Voting](draft-ecc-onchain-accountable-voting.md)

[^elgamal]: [T. ElGamal, "A public key cryptosystem and a signature scheme based on discrete logarithms", IEEE Transactions on Information Theory, vol. 31, no. 4, pp. 469-472, 1985](https://doi.org/10.1109/TIT.1985.1057074)

[^cp92]: [D. Chaum and T. P. Pedersen, "Wallet Databases with Observers", in Advances in Cryptology — CRYPTO '92, pp. 89-105, 1993](https://doi.org/10.1007/3-540-48071-4_7)

[^blake2]: [J.-P. Aumasson, S. Neves, Z. Wilcox-O'Hearn, and C. Winnerlein, "BLAKE2: simpler, smaller, fast as MD5", in Applied Cryptography and Network Security, pp. 119-135, 2013](https://doi.org/10.1007/978-3-642-38980-1_8)

[^ecies]: [V. Shoup, "A Proposal for an ISO Standard for Public Key Encryption", version 2.1, 2001](https://www.shoup.net/papers/iso-2_1.pdf)
