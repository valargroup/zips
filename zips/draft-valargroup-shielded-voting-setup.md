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

The key words "MUST" and "MAY" in this document are to be
interpreted as described in BCP 14 [^BCP14] when, and only when,
they appear in all capitals.

The terms below are to be interpreted as follows:

Vote chain
: The blockchain that serves as the single source of truth for voting
  operations. See [System Overview] for the state it maintains.

Voting round
: A complete instance of a coinholder vote, scoped to a single Zcash
  mainnet snapshot and a fresh Election Authority key.

Vote round ID
: A unique identifier for a voting round. See [Poll Creation] for the
  computation.

Poll runner
: The entity responsible for conducting a voting round.

Vote manager
: The on-chain role authorized to create voting rounds.

Bootstrap operator
: The entity that provisions the vote chain genesis and initial
  validator set.

Validator
: A vote chain consensus participant. See [Validator] under Roles for
  responsibilities and keypair details.

Bonded validator
: A validator whose stake is active under the standard Cosmos SDK
  `x/staking` module [^cosmos-staking]: its delegation is committed,
  it participates in consensus, and it is eligible to produce blocks.

Submission server
: An untrusted service that accepts encrypted vote share payloads
  from voters and submits the corresponding share reveal
  transactions to the vote chain. Specified in
  `draft-valargroup-submission-server` [^draft-submission-server].

Election Authority (EA)
: A virtual signing key, jointly constructed by validators during a key
  ceremony so that no single party holds the private key. Used to encrypt
  vote shares and decrypt the final tally. See
  `draft-valargroup-ea-key-ceremony` [^draft-ceremony] for the
  ceremony protocol.

Snapshot height
: The Zcash mainnet block height at which eligible Orchard note balances
  are captured. See [Snapshot Configuration] for constraints.

For definitions of cryptographic terms including *alternate nullifier*,
*nullifier non-membership tree*, *nullifier domain*, *pool snapshot*, and
*claim*, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
EA key ceremony terms, see `draft-valargroup-ea-key-ceremony`
[^draft-ceremony]. For PIR-related terms, see `draft-valargroup-gov-pir`
[^draft-pir].


# Abstract

This ZIP specifies how to operate the infrastructure for Zcash shielded
coinholder voting. It defines a purpose-built vote chain built on Cosmos
SDK, three operator roles (bootstrap operator, vote manager, validator),
and the lifecycle of a voting round from snapshot selection through tally
verification.

The vote chain stores vote commitments in a Poseidon Merkle tree, tracks
three nullifier sets to prevent double-voting, and accumulates encrypted
vote shares as homomorphic El Gamal ciphertexts. Each transaction
(delegation, vote, share reveal) is verified by a zero-knowledge proof
on-chain. Validators join through an automated onboarding script that
handles binary distribution, key generation, and on-chain registration.
A separate nullifier service provides private information retrieval of
exclusion proofs so voters can prove note non-spending without revealing
which notes they hold.

Tally correctness is independently verifiable: validators submit partial
decryptions which are Lagrange-combined on-chain. Any party with access
to the chain state can re-derive the combination from the stored
partials and confirm the decrypted result.


# Motivation

The Zcash Shielded Voting Protocol [^draft-voting-protocol] defines a
cryptographic protocol for private coinholder voting. This ZIP specifies
the deployment infrastructure required to run that protocol: the vote
chain, operator roles, and voting round lifecycle.


# Privacy Implications

- Zero-knowledge proofs and encryption hide the contents of
  delegations, votes, and share reveals, but not the network-layer
  metadata associated with their submission. A vote chain validator
  sees the source IP, submission timestamp, connection correlation,
  and P2P propagation pattern of every transaction it receives.
- PIR queries against the nullifier service are private in content
  but not in timing: a nullifier service operator sees the source IP
  and time of each query, which reveals that a given client is
  participating in the current voting round.
- The vote chain is a public ledger. Transaction contents are
  encrypted or zero-knowledge-proven, but their existence, ordering,
  and block-inclusion timing are a permanent public record.
- Bootstrap operators learn the network identities of validators
  during onboarding (see [Onboarding Validators]).
- Validator power distribution affects the trust model for the EA
  key ceremony. See `draft-valargroup-ea-key-ceremony`
  [^draft-ceremony] for EA-specific privacy implications.


# Requirements

- A new poll runner can set up infrastructure and conduct a voting round
  by following this specification and the referenced companion ZIPs.
- The vote chain operates as a public, verifiable ledger — anyone can run
  a monitoring node to audit.
- The system operates with partial validator availability.


# Non-requirements

- Governance policy decisions such as proposal eligibility, quorum
  requirements, and fund disbursement rules (see ZIP 1016 [^zip-1016]).


# Specification

## Proposals and Decisions

The voting process specified by this ZIP allows a poll runner to put
one or more questions — *proposals* — to eligible voters. For each
proposal, voters choose exactly one of a predefined set of labeled
*options*; the chosen option is the voter's *decision* for that
proposal.

### Structure of a proposal

Each proposal has:

- A **title**, short and human-readable.
- An optional **description** providing additional context.
- Between 2 and 8 **options**, each carrying a human-readable label.
  Option labels MUST be non-empty ASCII strings.

Proposals in a voting round are assigned 1-indexed sequential
identifiers; options within a proposal are assigned 0-indexed
sequential indices.

### Decisions

A **decision** is a voter's chosen option for a specific proposal,
represented as the option's 0-indexed position within that proposal's
option list. Decisions are recorded in the encrypted share
accumulator, keyed by `(proposal_id, vote_decision)`; see
`draft-valargroup-voting-protocol` [^draft-voting-protocol] for the
cryptographic construction.

### Kinds of polls that can be expressed

A voting round can carry **1 to 15 independent proposals**, and each
proposal can offer **2 to 8 labeled options**. This is sufficient for:

- Yes/no questions ("Approve proposal X?" with options Yes / No).
- Multiple-choice preference questions (for example, choosing among
  named candidates or funding tiers).
- Rating-style questions using a fixed option ladder.

The following ballot shapes are out of scope for this specification:

- Free-form write-in answers.
- Ranked-choice or weighted-ranking ballots.
- More than 15 proposals in a single voting round, or more than 8
  options in a single proposal.

The 15-proposal upper bound is imposed by the zero-knowledge vote
authority bitmask (1 bit is reserved as a sentinel). Polls requiring
more proposals or richer ballot structures are split across multiple
rounds or expressed through an external layer.

## System Overview

The coinholder voting system operates on a purpose-built Cosmos SDK vote
chain. Zcash mainnet snapshots provide the set of eligible Orchard note
balances.

The vote chain stores:

- A **Vote Commitment Tree** (VCT): a Poseidon Merkle tree of vote
  commitments.
- Three **nullifier sets**: governance nullifiers (alternate nullifiers
  from note claims), VAN nullifiers (from delegation consumption), and
  share nullifiers (from share reveals).
- An **encrypted share accumulator** per (proposal, decision): the
  homomorphic sum of El Gamal ciphertexts for each vote option.

The vote chain verifies a zero-knowledge proof for each transaction type:
delegation, vote, and share reveal. The proof circuits are specified in
`draft-valargroup-voting-protocol` [^draft-voting-protocol].

## Deployment Architecture

A complete deployment consists of:

- **Vote chain nodes** — one or more `svoted` instances running CometBFT
  consensus. Each `svoted` binary additionally includes the
  **submission server** (an untrusted service that accepts vote
  share payloads from clients and submits the corresponding share
  reveal transactions at client-specified times). The submission
  server shares the node's process but is functionally decoupled
  from chain consensus; see `draft-valargroup-submission-server`
  [^draft-submission-server].
- **Nullifier service** — a PIR server that provides private nullifier
  exclusion proofs to voters (see [Nullifier Service (PIR Server)]).
- **Service discovery API** — a centralized bootstrap directory that
  wallet clients and joining validators query to discover vote chain and
  PIR server endpoints (see [Service Discovery]).
- **Admin interface** — a management UI for approving pending validator
  registrations and configuring service discovery endpoints.

## Roles

### Bootstrap Operator

The bootstrap operator generates the vote chain genesis block and
provisions the initial validator set. At genesis, a single vote
manager account is created with a balance of the chain's native token
(denom `usvote`) sized to fund all planned validators. From that
account the bootstrap operator funds each validator via
`MsgAuthorizedSend`, a transfer message gated by the vote chain's
ante handler: the vote manager MAY send to any address, and bonded
validators MAY send to the vote manager or to other bonded
validators; all other transfers — including the standard Cosmos bank
`MsgSend` and `MsgMultiSend` messages — are rejected.

The amount transferred to each validator at bonding time determines
their consensus voting power. An even distribution across validators
reduces the risk of consensus capture. The bootstrap operator's
activities are confined to genesis. The keypair that controls the
genesis `vote_manager` address continues afterwards as the initial
vote manager (see [Vote Manager]), and MAY transfer that role via
`MsgSetVoteManager`.

### Vote Manager

The vote manager creates voting rounds by submitting
transactions with a `MsgCreateVotingSession`. Only the vote manager
role can publish new rounds.

The vote manager address is set in the genesis block (see
[Genesis Validator Setup]). The current vote manager MAY transfer
the role to another address via `MsgSetVoteManager`; the transfer
is atomic and moves the full account balance to the new address.
No other account can claim or reassign the role.

### Validator

Validators participate in consensus, the EA key ceremony (see
`draft-valargroup-ea-key-ceremony` [^draft-ceremony]), and automatic
tally computation. Each validator maintains three keypairs:

- **Consensus keypair**: used for CometBFT consensus.
- **Account keypair**: used for submitting chain transactions.
- **Pallas keypair**: used for ECIES key exchange during the EA ceremony.

Validators join the network via the automated `join.sh` [^join-sh]
script or by building from source. See [Onboarding Validators].

## Vote Chain Infrastructure

### Genesis Validator Setup

The bootstrap operator builds the vote chain binary (Go + Rust FFI for
Halo 2 and RedPallas verification), initializes a single-validator chain,
and starts the node.

Initialization generates a Cosmos validator key, a Pallas keypair for the
EA ceremony, and a genesis block with the chain ID `svote-1`. The node
exposes the standard CometBFT P2P, RPC, and Cosmos SDK REST endpoints.

After the chain is producing blocks, the bootstrap operator registers the
node's public URL in the service discovery layer (see [Service Discovery])
so that joining validators and wallet clients can find the network.

### Onboarding Validators

New validators join through `join.sh` [^join-sh], a self-contained
script that requires no local clone of the repository. The script:

1. Downloads pre-built `svoted` and `create-val-tx` binaries and
   verifies their SHA-256 checksums.
2. Discovers a live validator via the service discovery API.
3. Fetches genesis and syncs to the current height.
4. Generates consensus, account, and Pallas keypairs.
5. Self-registers with the service discovery API (appears as "pending"
   in the admin UI).
6. Waits for the bootstrap operator to approve and fund the validator
   via the admin UI.
7. On receiving funds, auto-registers on-chain with
   `MsgCreateValidatorWithPallasKey`.

The funding amount equals the validator's consensus voting power.
Developers with a local clone can alternatively run
`mise run validator:join`, which builds from source and then runs the
same `join.sh` flow.

### Nullifier Service (PIR Server)

The nullifier service provides nullifier exclusion proofs to voters via
PIR.

The service pipeline (each step has a corresponding
`mise run nullifier:<step>` task):

1. **Ingest**: fetch Orchard nullifiers from Zcash mainnet via a
   lightwallet server (`lightwalletd`), or download a pre-built snapshot.
2. **Export**: build the nullifier non-membership tree (Indexed Merkle
   Tree as specified in `draft-valargroup-orchard-balance-proof`
   [^draft-balance-proof]) and export the three-tier PIR database as
   specified in `draft-valargroup-gov-pir` [^draft-pir]. The exported
   files allow the server to restart without rebuilding the tree from
   raw nullifiers.
3. **Serve**: expose a query endpoint for voters to privately retrieve
   exclusion proofs.

### Service Discovery

The service discovery API is a centralized bootstrap directory that
serves as the entry point for both validator onboarding and wallet
client integration. It exposes a `/api/voting-config` endpoint that
returns:

- **Vote chain endpoints** — REST API URLs for active validators.
- **PIR server endpoints** — URLs for nullifier exclusion proof queries.

Joining validators query this API to discover a seed node, fetch its
CometBFT P2P identity and genesis, and connect. Once connected,
CometBFT's peer exchange (PEX) protocol handles discovery of
additional peers — the API is only needed for initial bootstrap.

Wallet clients query the same API to discover vote chain and PIR
server endpoints for voter-facing operations.

New validators register themselves with the API after joining. The
bootstrap operator approves pending registrations through an admin
interface, after which the validator appears in the published
endpoint list.

## Conducting a Voting Round

### Snapshot Configuration

The poll runner selects a Zcash mainnet snapshot height subject to these
constraints:

- The height MUST be at or after NU5 activation (Orchard is required).

Selecting the snapshot triggers:

1. The PIR server rebuilds its nullifier non-membership tree at that
   height.
2. The note commitment tree root ($\mathsf{nc\_root}$) and nullifier IMT
   root ($\mathsf{nullifier\_imt\_root}$) are captured at the snapshot
   height.

### Poll Creation

The vote manager publishes a new round via `MsgCreateVotingSession` with
the following parameters:

- `snapshot_height`: the selected Zcash mainnet block height.
- `snapshot_blockhash`: the block hash at `snapshot_height`.
- `proposals`: 1 to 15 proposals, each with 2 to 8 labeled options (e.g.,
  "Support" / "Oppose"). The limit is 15 because the circuit's
  proposal authority bitmask reserves bit 0 as a sentinel.
- `vote_end_time`: deadline for all voting phases.
- `nullifier_imt_root`: root of the nullifier non-membership tree at
  snapshot.
- `nc_root`: Orchard note commitment tree root at snapshot.
- `verification_keys`: verification keys for the ZKP circuits (delegation,
  vote, share reveal).

The vote round ID is a Poseidon hash of the round parameters
(snapshot height, block hash, proposals, end time, nullifier IMT root,
note commitment root). It is a Pallas field element because it enters
ZKP circuits as a public input.

The round enters the **PENDING** state. The EA key ceremony (see
`draft-valargroup-ea-key-ceremony` [^draft-ceremony]) runs
automatically. On successful completion, the
round transitions to **ACTIVE**, the voting window opens, and the
transition timestamp is recorded as `ceremony_phase_start`. Clients use
`ceremony_phase_start` together with `vote_end_time` to compute the
last-moment buffer for submission timing, as specified in the
"Last-Moment Buffer" section of `draft-valargroup-submission-server`
[^draft-submission-server-lmb].

### Round Lifecycle

1. **PENDING**: round created, awaiting EA key ceremony.
2. **ACTIVE**: ceremony complete, voting window open. Voters may delegate,
   vote, and submit shares (see `draft-valargroup-voting-protocol`
   [^draft-voting-protocol]).
3. **TALLYING**: `vote_end_time` has passed. Tally decryption runs
   automatically (see `draft-valargroup-ea-key-ceremony`
   [^draft-ceremony]).
4. **FINALIZED**: tally published and verifiable.

### Timing Parameters

| Parameter                      | Value        | Notes                         |
| ------------------------------ | ------------ | ----------------------------- |
| EA ceremony timing             |              | See [^draft-ceremony]         |
| Voting window                  | Configurable | Set by `vote_end_time`        |
| Tally computation              | Automatic    | Triggered after window closes |

## Verification and Auditing

The vote chain is publicly readable. Any party running a full node
of the chain — a validator, the vote manager, or an independent
observer — can verify all aspects of a voting round:

- Verify every zero-knowledge proof submitted in delegation, vote, and
  share reveal transactions.
- Track all three nullifier sets (governance, VAN, share) for
  double-spending.
- Recompute the aggregate El Gamal ciphertexts per (proposal, decision)
  from individual share reveals.
- Verify tally correctness by re-deriving the Lagrange combination of
  stored partial decryptions and confirming the decrypted result (see
  `draft-valargroup-ea-key-ceremony` [^draft-ceremony]).

Because the partial decryptions are stored on-chain, any party can
re-derive the Lagrange combination and check the final tally without
relying on a single validator's output.


# Rationale

**Separate vote chain (not Zcash mainnet)**: the vote chain is purpose-built
for governance with ZKP-optimized state transitions (Poseidon hashing, custom
transaction types). Zcash mainnet's transaction throughput and scripting model
are not designed for interactive multi-phase voting protocols.

**Orchard-only snapshots**: the voting protocol is built on Orchard's
circuit-friendly primitives (Poseidon hashing, Pallas curve). Sapling
and transparent pools use incompatible cryptographic constructions.
Coinholders who wish to participate can migrate funds to Orchard before
the snapshot.

**Cosmos SDK**: provides a mature BFT consensus engine (CometBFT),
validator lifecycle management (bonding, jailing for missed blocks or
missed ceremony acknowledgements, consensus power distribution), and a
transaction pipeline that can be extended with custom message types and
ante handlers for ZKP verification. The alternative — building a chain
from scratch — would duplicate well-tested consensus infrastructure.

**Funding equals voting power**: bonding serves three purposes: it
determines which validators participate in consensus and the EA key
ceremony (and thus can decrypt the tally), it enables jailing of
inactive validators who miss blocks or ceremony acknowledgements, and
an even funding split gives each validator a roughly equal probability
of becoming the block proposer — not important for correctness, but
important for liveness.

**Automated validator onboarding**: `join.sh` eliminates manual
coordination between the bootstrap operator and joining validators. The
self-registration, admin-approval, and auto-bonding flow allows the
network to grow without requiring validators to build from source or
understand Cosmos SDK tooling.

**Vote manager reassignment**: only the current vote manager can
transfer the role (via `MsgSetVoteManager`). This is a deliberate
single-party control: the vote manager can create rounds but cannot
forge votes, and the worst-case mitigation for a compromised vote
manager is to spin up a new chain.


# Open Issues

- **Role consolidation**: evaluate whether the bootstrap operator and
  vote manager concepts (already the same keypair at genesis; see
  [Bootstrap Operator]) merit separate treatment in the specification.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-voting-protocol.md)

[^draft-ceremony]: [Draft ZIP: Election Authority Key Ceremony](draft-valargroup-ea-key-ceremony.md)

[^draft-pir]: [Draft ZIP: Private Information Retrieval for Nullifier Exclusion Proofs](draft-valargroup-gov-pir.md)

[^draft-submission-server]: [Draft ZIP: Vote Share Submission Server](draft-valargroup-submission-server.md)

[^draft-submission-server-lmb]: [Draft ZIP: Vote Share Submission Server, Section: Last-Moment Buffer](draft-valargroup-submission-server.md#last-moment-buffer)

[^draft-onchain-voting]: [Draft ZIP: On-chain Accountable Voting](draft-ecc-onchain-accountable-voting.md)

[^join-sh]: [join.sh — validator join script](https://gist.github.com/greg0x/71bec808fbd02a7ef2a29b4386b8d842)

[^cosmos-staking]: [Cosmos SDK `x/staking` module documentation](https://docs.cosmos.network/main/build/modules/staking)
