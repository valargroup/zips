    ZIP: Unassigned
    Title: Zcash Signed Messages
    Owners: {to be filled in}
    Status: Draft
    Category: Standards / Wallet
    Created: 2026-11-15
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/???>


# Terminology

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this
document are to be interpreted as described in BCP 14 [^BCP14] when, and
only when, they appear in all capitals.

The character § is used when referring to sections of the Zcash Protocol
Specification. [^protocol]

The terms "*Orchard pool*" and "*Ironwood pool*" are to be interpreted as
described in ZIP 229. [^zip-0229] "Orchard-protocol keys" are the key components
of § 4.2.3 ‘Orchard Key Components’ as modified by ZIP 2005 [^zip-2005]; the same
keys and addresses are used for both pools.

The terms below are to be interpreted as follows:

Signer

: The software and/or hardware that holds Orchard-protocol spending key material,
  renders a message to its user, and produces a signature. Where a hardware
  wallet is involved, "the device" is the part that holds $\mathsf{ask}$ and has
  a trusted display, and "the host" is everything else.

Verifier

: Any party that checks a signature: an off-chain service, another wallet, or a
  contract, program, or module on another chain.

Signing identity

: A 32-byte *identity commitment* $\mathsf{idc}$ that commits to exactly one
  diversified Orchard-protocol payment address. It is the only signer identifier
  that appears in signed messages.

Disclosed identity / blinded identity

: The two ways of deriving $\mathsf{idc}$ from an address. A disclosed identity can
  be computed by anyone who knows the address. A blinded identity can be computed
  only by the holder of the address's message derivation key.

Identity certificate

: A zero-knowledge proof, made once per signing identity, that binds a *message
  verification key* $\mathsf{rk}$ to $\mathsf{idc}$.

Chain binding

: A profile of this ZIP for one family of chains, defining how that chain's
  transactions are identified, hashed, rendered, and verified.


# Abstract

This ZIP specifies a way for the holder of an Orchard-protocol spending key, as
used in the *Ironwood pool*, to sign human-readable messages and to authorize
transactions on other chains.

A signature has two layers. An *identity certificate* reuses the unmodified
Orchard Action circuit, applied to a fixed fake note, to bind a rerandomized
spend validating key $\mathsf{rk}$ to a commitment to one diversified payment
address. It needs to be verified only once per identity. A *message signature* is
then an ordinary $\mathsf{SpendAuthSig^{Orchard}}$ (RedPallas) signature under
$\mathsf{rk}$ over the digest of a typed envelope.

Identities derived from different diversified addresses of the same key are
unlinkable. Blinded identities remain unlinkable against an adversary that can
compute discrete logarithms.

The envelope has a small closed set of message kinds, each with mandatory display
rules. It deliberately has no way to sign an opaque digest. A signature of one
kind can never be accepted as another kind, as a Zcash transaction authorization,
or by a verifier on a different chain or account.


# Motivation

ZIP 304 [^zip-0304] defines signatures for Sapling addresses. There is no
equivalent for Orchard-protocol addresses, and ZIP 304 signs an unstructured byte
string with no domain separation between applications.

Meanwhile, account abstraction on other chains (ERC-1271 [^eip-1271] and
ERC-4337 [^erc-4337] contracts on EVM chains, programs on Solana, account modules
on Cosmos SDK chains) lets an account be controlled by any signature scheme its
verifier implements. A Zcash user could therefore control accounts on those
chains directly with their shielded key, without creating, backing up, and
securing a second key. This is useful for cross-chain swaps and intents that
start or end in the *Ironwood pool*, for paying a Zcash user on another chain
given only their shielded address, and for signing in to applications with a
shielded identity.

Every ecosystem that added message signing has had to repair it after the fact:
raw-hash signing that could authorize transactions, typed data that hardware
wallets could only show as a hash, "gasless" signatures that drained wallets,
sign-in messages replayed across sites. This ZIP starts from those lessons, which
are collected in [Lessons from prior art](#lessons-from-prior-art).


# Privacy Implications

* All signatures by one signing identity are linkable to each other. This is
  inherent: the identity is an account.
* Two identities derived from different diversified addresses are unlinkable to
  each other and to any Zcash transaction, to the extent set out in
  [Unlinkability](#unlinkability). For blinded identities this holds even against
  an adversary that can compute discrete logarithms on Pallas.
* A *disclosed* identity is computable from its address. Anyone who has been
  given that address (for example, anyone who has paid it) can recognise every
  account and signature that uses the identity. Disclosed identities SHOULD be
  used only when a counterparty needs to derive the identity from the address,
  and SHOULD use a diversified address that is not used for anything else.
* A holder of the *message derivation key* $\mathsf{mdk}$ who also knows an
  address can recognise that address's blinded identity. $\mathsf{mdk}$ is not
  part of any viewing key, so sharing a full viewing key with an auditor does not
  by itself reveal blinded identities.
* Activity on another chain is public on that chain. This ZIP hides which Zcash
  key controls an account; it does not hide what the account does.


# Requirements

* **Spend-key control.** Only a party able to make spend authorization signatures
  for an address can sign as an identity derived from that address. A viewing key
  is not sufficient.
* **Diversified identities.** Each diversified address yields its own identity.
  Identities from different addresses of one key are computationally unlinkable;
  blinded identities are unlinkable against discrete-logarithm-breaking
  adversaries.
* **No circuit changes.** Verification reuses the Action circuit and verifying
  key deployed for the *Ironwood pool*.
* **Cheap repeated verification.** After an identity has been certified once, each
  further signature costs one RedPallas verification and one BLAKE2b hash.
* **Hardware wallet and FROST compatibility.** The device needs only to parse and
  display a message, hash it, and make a rerandomized RedPallas signature, as it
  already does for transactions. Proving is done by the host. Rerandomized FROST
  [^zip-0312] can produce the signature.
* **What you see is what you sign.** Every byte that influences the digest is
  either displayed or is recomputed by the Signer from displayed data.
* **Domain separation.** A signature is valid for exactly one kind, one signing
  identity, and (for chain actions) one chain and one account.
* **Zcash funds are never at risk.** No output of this protocol can be used,
  whole or in part, to authorize a Zcash transaction.
* **Non-malleability.** Given a valid signature, a third party cannot produce a
  different valid signature for the same identity and message.


# Non-requirements

* Post-quantum *unforgeability*. A discrete-logarithm-breaking adversary can
  forge RedPallas signatures and Action proofs. See
  [Quantum adversaries](#quantum-adversaries).
* Hiding the link between two signatures by the same identity.
* Signing with Sapling or transparent keys.
* Verifying the native Action proof inexpensively on every chain. Chain bindings
  MAY transport the identity certificate in a wrapped form.


# Specification

## Notation

$a\,||\,b$ is concatenation. Integers are encoded little-endian. $\mathsf{u16}(n)$
and $\mathsf{u64}(n)$ are 2-byte and 8-byte encodings. $\mathsf{lp8}(s)$ is a
1-byte length followed by the bytes of $s$ (at most 255); $\mathsf{lp16}(s)$ uses
a 2-byte length (at most 65535).

$\mathsf{BLAKE2b\text{-}256}(p, x)$ and $\mathsf{BLAKE2b\text{-}512}(p, x)$ are
unkeyed BLAKE2b [^RFC7693] with 16-byte personalization $p$.

The following are as defined in the protocol specification [^protocol] for the
Orchard protocol, including the changes for NU6.3: $\mathsf{DiversifyHash^{Orchard}}$,
$\mathsf{NoteCommit^{Orchard}}$, $\mathsf{ValueCommit^{Orchard}}$,
$\mathsf{DeriveNullifier}$, $\mathsf{MerkleCRH^{Orchard}}$,
$\mathsf{MerkleDepth^{Orchard}}$, $\mathsf{Uncommitted^{Orchard}}$,
$\mathsf{Extract}_{\mathbb{P}}$, $\mathsf{repr}_{\mathbb{P}}$,
$\mathsf{ToBase^{Orchard}}$, $\mathsf{ToScalar^{Orchard}}$, $\mathsf{PRF^{expand}}$,
$\mathsf{SpendAuthSig^{Orchard}}$, and the Action statement (§ ‘Action Statement (Orchard)’) with primary
input $(\mathsf{rt}, \mathsf{cv^{net}}, \mathsf{nf^{old}}, \mathsf{rk}, \mathsf{cm}_x, \mathsf{enableSpends}, \mathsf{enableOutputs}, \mathsf{disableCrossAddress})$.

Define, for a tag byte $t$, a 32-byte key $k$, and a byte string $x$:

$$\mathsf{H^{SM}}_t(k, x) = \mathsf{BLAKE2b\text{-}512}(\texttt{“Zcash\_SM\_Derive\_”},\; [t]\,||\,k\,||\,x)$$

## Overview

```
 diversified address (d, pk_d)
        │  seed: public (disclosed) or derived from mdk (blinded)
        ▼
 fake note, value 1  ──►  idc = root of a tree holding only that note
        │
        │  Action proof, made once          ┌──────────────────────────┐
        ├──────────────────────────────────►│ identity certificate     │
        │                                   │ binds rk = ak + [α]G     │
        │                                   │ to idc                   │
        │                                   └──────────────────────────┘
        │  RedPallas, every message         ┌──────────────────────────┐
        └──────────────────────────────────►│ σ over digest(envelope)  │
                                            └──────────────────────────┘
```

## Message derivation key

Let $\mathsf{sk}$ be the Orchard-protocol spending key. Define

$$\mathsf{mdk} = \mathsf{truncate}_{32}\big(\mathsf{PRF^{expand}_{sk}}([\mathtt{0x0E}])\big)$$

$\mathsf{mdk}$ is needed to derive blinded identities and to create identity
certificates. It is not needed to verify anything. A hardware wallet exports
$\mathsf{mdk}$ to its host alongside the full viewing key, with the same user
consent.

## Signing identities

A signing identity is derived from an external-scope diversified payment address
$(\mathsf{d}, \mathsf{pk_d})$. Internal (change) addresses MUST NOT be used.

Let $\mathsf{g_d} = \mathsf{DiversifyHash^{Orchard}}(\mathsf{d})$ and let
$\mathsf{addr} = \mathsf{d}\,||\,\mathsf{LEBS2OSP}_{256}(\mathsf{repr}_{\mathbb{P}}(\mathsf{pk_d}))$
be the 43-byte raw encoding of the Orchard receiver. [^zip-0316]

1. Derive the 32-byte identity seed:
   * disclosed: $\mathsf{seed} = \mathsf{BLAKE2b\text{-}256}(\texttt{“Zcash\_SM\_PubSeed”}, \mathsf{addr})$
   * blinded: $\mathsf{seed} = \mathsf{truncate}_{32}\big(\mathsf{H^{SM}_{\mathtt{0x01}}}(\mathsf{mdk}, \mathsf{addr})\big)$
2. Let $\text{ρ}^\star = \mathsf{ToBase^{Orchard}}(\mathsf{H^{SM}_{\mathtt{0x02}}}(\mathsf{seed}, \varepsilon))$,
   $\text{ψ}^\star = \mathsf{ToBase^{Orchard}}(\mathsf{H^{SM}_{\mathtt{0x03}}}(\mathsf{seed}, \varepsilon))$, and
   $\mathsf{rcm}^\star = \mathsf{ToScalar^{Orchard}}(\mathsf{H^{SM}_{\mathtt{0x04}}}(\mathsf{seed}, \varepsilon))$.
3. Let $\mathsf{cm}^\star = \mathsf{NoteCommit^{Orchard}_{rcm^\star}}(\mathsf{repr}_{\mathbb{P}}(\mathsf{g_d}), \mathsf{repr}_{\mathbb{P}}(\mathsf{pk_d}), 1, \text{ρ}^\star, \text{ψ}^\star)$.
   If $\mathsf{cm}^\star = \bot$ the address cannot be used as an identity.
4. Let $\mathsf{idc}$ be the root of the Merkle tree of depth
   $\mathsf{MerkleDepth^{Orchard}}$, hashed with $\mathsf{MerkleCRH^{Orchard}}$,
   that has $\mathsf{Extract}_{\mathbb{P}}(\mathsf{cm}^\star)$ at position 0 and
   $\mathsf{Uncommitted^{Orchard}}$ at every other position.

The fake note $(\mathsf{d}, \mathsf{pk_d}, 1, \text{ρ}^\star, \text{ψ}^\star, \mathsf{rcm}^\star)$
is never a note of any pool.

The human-readable form of $\mathsf{idc}$ is its Bech32m [^bip-0350] encoding
with Human-Readable Part $\texttt{“zsid”}$.

An *opening* of an identity is $\mathsf{addr}$ for a disclosed identity, or
$(\mathsf{addr}, \mathsf{seed})$ for a blinded identity. Anyone given an opening
can recompute $\mathsf{idc}$ by steps 2 to 4 and so confirm which address the
identity commits to. Revealing the opening of a blinded identity is permanent and
transferable, and reduces that identity to the privacy of a disclosed one.

Wallets SHOULD allocate identities at sequential diversifier indices from a fixed
starting index that is not used for payment addresses, so that identities can be
rediscovered from the seed phrase.

## Identity certificate

### Creation

The inputs are the address, its identity seed, $\mathsf{mdk}$, and the full
viewing key components $(\mathsf{ak}^{\mathbb{P}}, \mathsf{nk}, \mathsf{rivk})$.
No spend authorizing key is needed.

1. Compute $\text{ρ}^\star, \text{ψ}^\star, \mathsf{rcm}^\star, \mathsf{cm}^\star, \mathsf{idc}$
   as above, and the Merkle path $\mathsf{path}$ from position 0 to $\mathsf{idc}$.
2. Let $\alpha = \mathsf{ToScalar^{Orchard}}(\mathsf{H^{SM}_{\mathtt{0x05}}}(\mathsf{mdk}, \mathsf{idc}))$,
   $\text{ψ}' = \mathsf{ToBase^{Orchard}}(\mathsf{H^{SM}_{\mathtt{0x06}}}(\mathsf{mdk}, \mathsf{idc}))$, and
   $\mathsf{rcm}' = \mathsf{ToScalar^{Orchard}}(\mathsf{H^{SM}_{\mathtt{0x07}}}(\mathsf{mdk}, \mathsf{idc}))$.
3. Let $\mathsf{nf}^\star = \mathsf{DeriveNullifier_{nk}}(\text{ρ}^\star, \text{ψ}^\star, \mathsf{cm}^\star)$.
4. Let $\mathsf{rk} = \mathsf{SpendAuthSig^{Orchard}.RandomizePublic}(\alpha, \mathsf{ak}^{\mathbb{P}})$.
5. Let $\mathsf{cm}_x' = \mathsf{Extract}_{\mathbb{P}}\big(\mathsf{NoteCommit^{Orchard}_{rcm'}}(\mathsf{repr}_{\mathbb{P}}(\mathsf{g_d}), \mathsf{repr}_{\mathbb{P}}(\mathsf{pk_d}), 0, \mathsf{nf}^\star, \text{ψ}')\big)$.
   This is a zero-valued output to the same address.
6. Let $\mathsf{cv}^\star = \mathsf{ValueCommit^{Orchard}_0}(1)$. This is a constant.
7. Let $\pi$ be an Action proof for the *Ironwood pool* circuit with primary input
   $$(\mathsf{idc},\; \mathsf{cv}^\star,\; \mathsf{nf}^\star,\; \mathsf{rk},\; \mathsf{cm}_x',\; \mathsf{enableSpends} = 1,\; \mathsf{enableOutputs} = 0,\; \mathsf{disableCrossAddress} = 1)$$
   and auxiliary input consisting of $\mathsf{path}$, position 0, the fake note,
   $\alpha$, $\mathsf{ak}^{\mathbb{P}}$, $\mathsf{nk}$, $\mathsf{rivk}$, the output
   note of step 5, and $\mathsf{rcv} = 0$.

The certificate is

$$\mathsf{cert} = \mathsf{nf}^\star\,||\,\mathsf{LEBS2OSP}_{256}(\mathsf{repr}_{\mathbb{P}}(\mathsf{rk}))\,||\,\mathsf{cm}_x'\,||\,\pi$$

which is $32 + 32 + 32 + 4992 = 5088$ bytes. It is the same for every signature
by the identity and MAY be cached and published.

### Verification

Given $\mathsf{idc}$ and $\mathsf{cert}$, a Verifier MUST:

1. Parse $\mathsf{cert}$, rejecting non-canonical encodings of $\mathsf{nf}^\star$,
   $\mathsf{rk}$, and $\mathsf{cm}_x'$, and rejecting $\mathsf{rk}$ if it is the
   identity point.
2. Verify $\pi$ as an Action proof with exactly the primary input of step 7 above,
   using the constant $\mathsf{cv}^\star$ and the constant flag values.

A Verifier MUST NOT treat a certificate as evidence of control on its own. A
certificate MUST be accepted only together with a valid message signature under
its $\mathsf{rk}$: either the message being verified, or a `Register` message
(see below) when an identity is being recorded for later use.

A Verifier that has recorded $(\mathsf{idc}, \mathsf{rk})$ after such a check
need not verify the certificate again.

## Message signature

Let $\mathsf{msg}$ be an encoded envelope as defined in [Envelope](#envelope),
whose first field is $\mathsf{idc}$.

$$\mathsf{digest} = \mathsf{BLAKE2b\text{-}256}(\texttt{“ZcashSignedMsgV1”},\; \mathsf{msg})$$

To sign, let $\mathsf{rsk} = \mathsf{SpendAuthSig^{Orchard}.RandomizePrivate}(\alpha, \mathsf{ask})$
and $\sigma = \mathsf{SpendAuthSig^{Orchard}.Sign}(\mathsf{rsk}, \mathsf{digest})$.
The device SHOULD derive $\alpha$ itself from $\mathsf{mdk}$ and $\mathsf{idc}$
rather than accept it from the host.

To verify, check
$\mathsf{SpendAuthSig^{Orchard}.Validate}(\mathsf{rk}, \mathsf{digest}, \sigma) = 1$,
where $\mathsf{rk}$ is the key certified for the $\mathsf{idc}$ in $\mathsf{msg}$,
and then apply the acceptance rules for the message kind.

## Envelope

$$\mathsf{msg} = \mathsf{idc}\,||\,[\mathsf{kind}]\,||\,\mathsf{body}$$

| `kind` | Name          | Purpose |
|--------|---------------|---------|
| `0x01` | `Text`        | A statement in plain text. Carries no authority over assets. |
| `0x02` | `SignIn`      | Authentication to an off-chain service. Carries no authority over assets. |
| `0x03` | `ChainAction` | Authorization of one transaction or typed message on another chain. |
| `0x04` | `Register`    | Proof of possession when recording $(\mathsf{idc}, \mathsf{rk})$ with a Verifier. |

All other values are reserved. A Signer MUST refuse to sign, and a Verifier MUST
reject, a kind it does not implement. Trailing bytes after $\mathsf{body}$ are
invalid.

### `Text`

| Field    | Encoding | Constraints |
|----------|----------|-------------|
| `domain` | `lp8`    | The RFC 3986 [^RFC3986] authority of the requesting application, lowercase ASCII, or empty if there is none. |
| `format` | 1 byte   | `0x00`: bytes `0x20`..`0x7E` and `0x0A` only. `0x01`: well-formed UTF-8 in NFC, with no characters of general category Cc other than `0x0A`, and no bidirectional control characters. |
| `text`   | `lp16`   | Non-empty. |

### `SignIn`

The fields and their meanings follow CAIP-122 [^caip-122].

| Field       | Encoding | Constraints |
|-------------|----------|-------------|
| `domain`    | `lp8`    | Non-empty RFC 3986 authority of the service, lowercase ASCII. |
| `uri`       | `lp16`   | Non-empty RFC 3986 URI, ASCII. |
| `statement` | `lp16`   | As `Text` format `0x00`, without `0x0A`. May be empty. |
| `nonce`     | `lp8`    | At least 8 ASCII alphanumeric characters, chosen by the service. |
| `issuedAt`  | `u64`    | Seconds since the Unix epoch. |
| `expiresAt` | `u64`    | Seconds since the Unix epoch. MUST be greater than `issuedAt`. |
| `resources` | 1-byte count, then `lp16` each | ASCII URIs. |

### `ChainAction`

| Field         | Encoding  | Constraints |
|---------------|-----------|-------------|
| `chain`       | `lp8`     | CAIP-2 [^caip-2] chain identifier, ASCII. |
| `account`     | `lp8`     | The account being authorized, in the chain's native binary form as fixed by the chain binding. |
| `expiresAt`   | `u64`     | Seconds since the Unix epoch. MUST be nonzero. |
| `encoding`    | `u16`     | Payload encoding, from the chain binding's registry. |
| `payloadHash` | 32 bytes  | The chain-native signing hash of the payload, as fixed by the chain binding for this `encoding`. |

`payloadHash` is the only field of any kind that is not displayed directly. A
Signer MUST compute it from the complete payload, which it MUST render as
required by the chain binding.

### `Register`

| Field     | Encoding | Constraints |
|-----------|----------|-------------|
| `chain`   | `lp8`    | As for `ChainAction`, or empty for an off-chain Verifier. |
| `account` | `lp8`    | As for `ChainAction`, or the Verifier's `domain` if `chain` is empty. |
| `rk`      | 32 bytes | The message verification key being registered. |

## Signer requirements

These rules are what prevents this ZIP from reintroducing blind signing. They
apply to the component with the trusted display; for a hardware wallet, that is
the device, and the host is assumed to be malicious.

1. A Signer MUST NOT expose any interface that signs a caller-supplied digest,
   envelope hash, or `payloadHash` with $\mathsf{rsk}$ or with $\mathsf{ask}$. It
   MUST construct $\mathsf{msg}$ and $\mathsf{digest}$ itself from the fields it
   displays.
2. Before signing, a Signer MUST display the message kind, using the headings
   below, and every field listed for that kind, in order:

   | Kind          | Heading | Fields displayed |
   |---------------|---------|------------------|
   | `Text`        | "Sign message" | signing identity; `domain` (or "no application"); `text` in full |
   | `SignIn`      | "Sign in" | `domain`; signing identity; `statement`; `uri`; `issuedAt`; `expiresAt`; each resource. `nonce` MAY be omitted from display. |
   | `ChainAction` | "Authorize transaction" | chain name and `chain`; `account`; signing identity; `expiresAt`; the payload, rendered per the chain binding |
   | `Register`    | "Register signing key" | signing identity; `chain`; `account` |

   Times are displayed as UTC date and time.
3. A Signer MUST reject a message it cannot display completely and faithfully.
   This includes a `Text` whose characters it cannot render, a `ChainAction`
   whose `chain`, `encoding`, or payload contents it cannot decode, and any
   message exceeding its display limits. There is no fallback that shows a hash
   in place of content.
4. The heading for `ChainAction` MUST be visually distinct from those of `Text`
   and `SignIn`, and MUST NOT be described to the user as signing a "message".
5. A Signer that knows the origin of the requesting application (for example, a
   browser wallet) MUST refuse a `Text` or `SignIn` whose `domain` does not match
   that origin, and MUST refuse a `SignIn` whose `issuedAt` is in the future or
   whose `expiresAt` is in the past.
6. A Signer MUST NOT rely on transaction simulation, or on labels supplied by the
   requesting application, in place of rendering the signed fields. Such
   information MAY be shown in addition, clearly marked as unverified.
7. A Signer MUST NOT use $\mathsf{ask}$ or any key derived from it by
   rerandomization to sign anything other than a Zcash transaction SIGHASH or a
   $\mathsf{digest}$ as defined in this ZIP.

## Verifier requirements

1. A Verifier MUST reject a message whose $\mathsf{idc}$ is not the identity it
   has a certified $\mathsf{rk}$ for.
2. A Verifier that controls assets or permissions MUST accept only `ChainAction`
   messages whose `chain` and `account` equal its own chain identifier and
   account, whose `expiresAt` has not passed, and whose `encoding` it implements.
   It MUST NOT accept `Text`, `SignIn`, or `Register` messages as authorization
   for any state change other than, for `Register`, recording $(\mathsf{idc}, \mathsf{rk})$.
3. A Verifier MUST provide replay protection for `ChainAction` messages. Where
   the payload does not already contain a nonce consumed by the account, the chain
   binding MUST add one.
4. A `SignIn` Verifier MUST check `domain` against its own authority, MUST check
   `issuedAt` and `expiresAt` against its clock, and MUST accept each `nonce` at
   most once.
5. A Verifier MUST identify a signed message by $(\mathsf{idc}, \mathsf{digest})$
   and not by the bytes of a signature or certificate.

## Detached signature encoding

For use outside a chain binding, a signature is encoded as

$$[\mathsf{flags}]\,||\,\mathsf{idc}\,||\,\mathsf{cert}^?\,||\,\sigma$$

where bit 0 of $\mathsf{flags}$ is 1 if and only if $\mathsf{cert}$ is present,
and all other bits are 0. The textual form is $\texttt{“zsm1:”}$ followed by the
Base64 encoding [^RFC4648] of those bytes. The envelope is conveyed separately.

To prove control of a payment address to a counterparty, a wallet signs a `Text`
or `SignIn` message with the address's disclosed identity and sends the address,
the envelope, and the detached signature with certificate. The counterparty
recomputes $\mathsf{idc}$ from the address.

## Chain bindings

A chain binding MUST specify:

* the CAIP-2 namespace it covers and the binary form of `account`;
* a registry of `encoding` values, and for each one the payload format, how
  `payloadHash` is computed from it, and the rendering rules a Signer follows;
* how a Verifier on that chain obtains `payloadHash` and `expiresAt`, enforces
  replay protection, and checks $\sigma$;
* how the identity certificate is verified on that chain, including any wrapped
  proof format, and how the account address is derived from $\mathsf{idc}$.

A binding SHOULD reuse the chain's existing signing hash and rendering
conventions unchanged, so that hardware wallet applications for that chain can
reuse their parsers and add only the envelope.

The bindings sketched below are provisional and are expected to be completed in
this ZIP or split into companion ZIPs.

### EVM chains (`eip155`)

`account` is the 20-byte address of a contract account that stores
$(\mathsf{idc}, \mathsf{rk})$. It can be an ERC-4337 account [^erc-4337], a
validator module for a modular account, or the delegate of an EIP-7702
[^eip-7702] account. Its address SHOULD be a deterministic function of
$\mathsf{idc}$ so that it can be funded before deployment.

| `encoding` | Payload | `payloadHash` | Rendering |
|------------|---------|---------------|-----------|
| `0x0001`   | EIP-712 typed data [^eip-712] | `keccak256(0x1901 ‖ domainSeparator ‖ hashStruct(message))` | Domain name, verifying contract, and every message field; ERC-7730 [^erc-7730] descriptors MAY be used for labels and formatting. |
| `0x0002`   | ERC-4337 user operation | `userOpHash` for the EntryPoint in use | Sender, each call's target, value, and decoded calldata, fee limits. |

The contract's ERC-1271 `isValidSignature(hash, signature)` and its user
operation validation reconstruct $\mathsf{msg}$ from their own chain id and
address, the `hash` they are given, and the `encoding` and `expiresAt` carried in
`signature`, then check $\sigma$ against the stored $\mathsf{rk}$. Binding the
account address into $\mathsf{msg}$ gives the cross-account replay protection of
ERC-7739 [^erc-7739]. BLAKE2b is available through the EIP-152 precompile
[^eip-152]. There is no precompile for Pallas, so RedPallas verification is done
in contract code, at a cost comparable to contract-level P-256 verification.

Verifying a Halo 2 Action proof directly in the EVM is not practical. The
certificate is expected to be checked through a succinct proof, over a curve the
EVM supports, that $\pi$ verifies for the given primary input. The certificate
contains no secrets, so producing that wrapped proof can be delegated.

EIP-191 `personal_sign` payloads [^eip-191] are intentionally not given an
`encoding`. Applications wanting a plain-text signature use `Text` or `SignIn`.

### Solana (`solana`)

Solana has no account abstraction at the protocol level, so `account` is the
32-byte address of a program-derived account whose controlling program verifies
$\sigma$. The payload is the ordered list of instructions that the program will
invoke on behalf of that account, and `payloadHash` is the SHA-256 hash of their
canonical serialization together with a nonce held in the account. Rendering
follows the rules a Solana wallet applies to the same instructions, and a Signer
MUST reject instructions for programs it cannot decode.

### Cosmos SDK chains (`cosmos`)

`account` is the account's address bytes. The payload is the transaction, and
`payloadHash` is the SHA-256 hash of its `SIGN_MODE_TEXTUAL` sign bytes
[^adr-050], which are an encoding of the screens shown to the user. Rendering is
exactly those screens.


# Rationale

## Lessons from prior art

### Ethereum

| Precedent | What happened | Decision here |
|-----------|---------------|---------------|
| `eth_sign` | Signs any 32-byte value, including a transaction hash. The user sees hex. It became a standard drainer tool and wallets disabled it. | Signer requirement 1: no digest-signing interface exists, at any layer. |
| EIP-191 `personal_sign` [^eip-191] | A `0x19` lead byte and text prefix guarantee the signed bytes are never a valid transaction. But there is no application or chain binding, and wallets disagree on showing bytes as text or hex. | A dedicated BLAKE2b personalization separates messages from Zcash SIGHASHes. `Text` has an explicit `format` and a `domain`. |
| EIP-155 [^eip-155] | Transactions were replayable across chains until the chain id was signed. | `chain` is mandatory in `ChainAction`. |
| EIP-712 [^eip-712] | Typed data with a domain separator. Every domain field is optional, so many deployments omitted the chain id or contract. The JSON form is awkward to parse on constrained devices, and for years hardware wallets displayed only the domain and message hashes. | Mandatory, fixed-layout, length-prefixed envelope fields that can be parsed and hashed in one pass. EIP-712 is kept as a *payload* so existing device parsers are reused, but a Signer that cannot decode it must refuse (requirement 3). |
| EIP-2612 permits, Permit2, marketplace orders [^eip-2612] | "Gasless" typed signatures that look like harmless messages but transfer token approvals became a leading cause of theft. | Kinds are a closed set. Anything that can move assets is a `ChainAction` with its own heading (requirement 4), and asset-controlling Verifiers accept nothing else (Verifier requirement 2). |
| EIP-4361 Sign-In with Ethereum [^eip-4361], CAIP-122 [^caip-122] | Sign-in is safe only if the wallet checks the requesting origin against the signed domain. Because it is layered on `personal_sign`, wallets that do not parse the text do not check. | `SignIn` is a distinct structured kind, so a Signer cannot handle it without seeing `domain` (requirement 5). Expiry is mandatory. |
| ERC-1271 [^eip-1271], ERC-7739 [^erc-7739] | Contract accounts can verify any scheme, which is the hook this ZIP uses. A signature valid for one account was replayable on another account with the same owner. | $\mathsf{idc}$ and `account` are both inside the signed envelope. |
| ERC-7730 clear signing [^erc-7730] | Readable display needs per-contract metadata, which itself has to be trusted. | Descriptors may improve labels but never replace display of the signed values (requirement 6). |
| Signature malleability (EIP-2) | Two encodings of one ECDSA signature broke systems that used signatures as identifiers. | RedPallas signatures are non-malleable; Verifier requirement 5 covers the certificate. |
| Multisig incidents in which signers approved a hash on a hardware wallet while a compromised interface showed a different transaction | The device could not show what was being signed. | The device builds the digest from what it displays and the host is treated as malicious. |

### Solana

| Precedent | What happened | Decision here |
|-----------|---------------|---------------|
| Transaction format | Instruction data is opaque bytes defined by each program. Wallets fall back to simulation, which malicious programs can make state-dependent, and hardware wallets fall back to blind signing. | Requirement 6 on simulation, and no blind-signing fallback. The Solana binding has the Signer reject undecodable instructions. |
| Off-chain message signing [^solana-offchain] | A `0xFF` lead byte makes the message an invalid transaction. A header carries a version, an application domain, the signers, and a *format* that states whether a hardware wallet can display the content. | The envelope follows the same shape: version in the personalization, $\mathsf{idc}$, `domain`, and `format` values tied to display capability. |
| Sign In With Solana [^siws] | Moved sign-in from free text to a structured request the wallet constructs. | Same as for EIP-4361. |
| Recent blockhash and durable nonces | Ordinary transactions expire quickly. Durable-nonce transactions do not, and were used to obtain signatures that were submitted much later. | `expiresAt` is mandatory in `ChainAction` and `SignIn` and is displayed. |

### Cosmos

| Precedent | What happened | Decision here |
|-----------|---------------|---------------|
| Amino JSON sign mode | The device shows the canonical JSON that is signed. Faithful, but raw. | Display is of typed fields with fixed headings rather than a serialization. |
| `SIGN_MODE_DIRECT` | Compact protobuf sign bytes that a device cannot interpret without the schema. | Not offered as a Cosmos payload encoding. |
| `SIGN_MODE_TEXTUAL`, ADR-050 [^adr-050] | The sign bytes *are* the screens shown to the user, and the chain re-renders the transaction to check them. | The envelope applies the same principle: displayed fields are the signed fields. The Cosmos binding uses Textual directly. |
| ADR-036 [^adr-036] | Off-chain messages are wrapped in a transaction with an empty chain id and zero sequence, so the existing signing path is reused and the result can never be broadcast. | The certificate applies the same idea to the proof (below). |

### Bitcoin and Zcash

BIP-322 [^bip-0322] signs messages by building a virtual transaction that spends
a nonexistent output. ZIP 304 [^zip-0304] does the same for Sapling by proving a
spend of a fake note in an otherwise empty tree. This ZIP inherits that
construction for the certificate, and adds what ZIP 304 lacks: a structured
envelope, a reusable certificate, and an identity that need not reveal the
address.

## Why a certificate plus a signature

ZIP 304 makes a fresh proof for every signature and binds the proof bytes into
the signed digest. That is too expensive for a chain to verify per transaction,
and it prevents the proof from being re-wrapped for chains that cannot verify
Halo 2 directly. Fixing $\alpha$ per identity makes $\mathsf{rk}$ a stable key,
so the proof says nothing about any message and can be checked once.

The certificate can be created with the full viewing key and $\mathsf{mdk}$
alone. That is why it is never accepted without a signature under $\mathsf{rk}$:
otherwise a viewing-key holder could register a certificate with a randomizer of
their choosing and deny the spender use of the account. The circuit does not
force $\alpha$ to be unique, so a Verifier records the first $\mathsf{rk}$
registered with proof of possession.

## Why the fake note is constructed this way

* The value is 1 because the Action circuit skips the Merkle path check for
  zero-valued notes, and the Merkle root is the only primary input that binds the
  address. Fixing $\mathsf{cv^{net}}$ to $\mathsf{ValueCommit_0}(1)$ and
  $\mathsf{enableOutputs}$ to 0 forces the spent value to be exactly 1.
* $\text{ρ}^\star$ depends on the address. With a constant $\text{ρ}$, the point
  $[\mathsf{PRF^{nfOrchard}_{nk}}(\text{ρ}) + \text{ψ}]\,\mathcal{K}^{\mathsf{Orchard}}$
  recoverable from $\mathsf{nf}^\star$ and $\mathsf{cm}^\star$ would be the same
  for every address of a key, linking them. ZIP 304 did not face this because a
  Sapling nullifier depends on the note commitment.
* The output is to the same address so that the proof is valid whatever the value
  of the cross-address flag, whose circuit semantics are not yet published.
  [^zip-2006]
* Using the Merkle root itself as the identity means Verifiers never compute
  Sinsemilla hashes; only a party checking an opening does.

## Why disclosed and blinded identities

A disclosed identity serves the ZIP 304 use case (prove control of this address)
and lets someone derive a recipient's account on another chain from their
shielded address. A blinded identity serves the common case in which nobody needs
to know the address. Both are the same statement to the circuit and to a
Verifier; they differ only in who can compute the seed.

$\mathsf{mdk}$ is derived from $\mathsf{sk}$ and not from the full viewing key so
that disclosing a viewing key does not disclose the holder's activity on other
chains.

## Why the Zcash network is not in the envelope

Keys for Mainnet and Testnet already differ through the ZIP 32 coin type
[^zip-0032], and $\mathsf{idc}$ does not otherwise depend on the network.


# Security Considerations

## Zcash funds

A message signature is over a BLAKE2b-256 digest personalized
$\texttt{“ZcashSignedMsgV1”}$. Transaction SIGHASHes are personalized
$\texttt{“ZcashTxHash\_”}$ followed by the consensus branch id [^zip-0244], so no
digest is also a SIGHASH. Signer requirements 1 and 7 keep it that way on the
device.

A certificate cannot be used as an Action of a transaction. Its anchor is the
root of a tree containing only the fake note, which is not an anchor of the
*Ironwood* or *Orchard* note commitment trees, and its nullifier and commitment
refer to notes that do not exist.

## Kind confusion

The kind byte and $\mathsf{idc}$ are in every digest. A `Text` or `SignIn`
signature therefore cannot satisfy a Verifier expecting a `ChainAction`, however
its text is chosen, and a `ChainAction` for one chain or account cannot satisfy
another.

## Unlinkability

For two identities derived from different addresses of one key, a Verifier sees
$\mathsf{idc}$, $\mathsf{nf}^\star$, $\mathsf{rk}$, $\mathsf{cm}_x'$, $\pi$, and
signatures.

* *Disclosed identities, classical adversary.* The addresses themselves are
  unlinkable under the same assumption as for Zcash payments.
  $\mathsf{nf}^\star$ reveals at most a PRF output on an address-specific input.
  $\mathsf{rk}$ is $\mathsf{ak}^{\mathbb{P}}$ masked by a pseudorandom $\alpha$,
  and $\mathsf{cm}_x'$ is hidden by a pseudorandom $\mathsf{rcm}'$.
* *Blinded identities, discrete-logarithm-breaking adversary.* The values
  $\mathsf{rcm}^\star$, $\text{ψ}^\star$, $\alpha$, and $\mathsf{rcm}'$ are
  outputs of BLAKE2b keyed by $\mathsf{mdk}$. $\mathsf{idc}$ is then a hiding
  commitment; the scalar recoverable from $\mathsf{nf}^\star$ is masked by
  $\text{ψ}^\star$; $\mathsf{rk}$ and the signing key $\mathsf{rsk}$ recoverable
  from it are masked by $\alpha$. None of these depends on the hardness of
  discrete logarithms. This argument additionally requires the Action proof to be
  statistically zero-knowledge.
* A discrete-logarithm-breaking adversary who knows two addresses can link them,
  as it can today. It can therefore link disclosed identities, and blinded
  identities whose openings have been revealed.

## Quantum adversaries

Unforgeability rests on discrete logarithms in Pallas. Unlike a spend
authorization key in a shielded transaction, the $\mathsf{rk}$ of an account on
another chain is long-lived and public, so an adversary able to compute discrete
logarithms could take over every account controlled through this ZIP. Assets on
other chains are not covered by the Recovery Protocol of ZIP 2005 [^zip-2005].
Forging a signature does not reveal $\mathsf{ask}$, which stays masked by
$\alpha$.

## Certificate wrapping

A chain binding that accepts a wrapped certificate relies on the soundness of the
wrapping proof system and on the wrapped statement fixing the Action verifying
key and all eight primary inputs.


# Deployment

This ZIP requires no consensus change. It depends on the *Ironwood pool* Action
circuit and verifying key deployed with NU6.3. [^zip-0258]


# Reference implementation

TBD. Test vectors for identity derivation, certificates, and each envelope kind
are required before this ZIP can leave Draft.


# Open issues

* Allocation of the $\mathsf{PRF^{expand}}$ domain byte $\mathtt{0x0E}$ in
  § 4.1.2, and of the BLAKE2b personalizations used here.
* Confirmation that Action circuit implementations accept $\mathsf{rcv} = 0$ (as
  ZIP 304 relies on for Sapling). If not, use $\mathsf{rcv} = 1$ and the
  corresponding constant $\mathsf{cv}^\star$.
* The exact constraint imposed by $\mathsf{disableCrossAddress}$, pending
  ZIP 2006, and whether fixing it to 1 is the right choice.
* A reviewed statement of the zero-knowledge property of Action proofs against
  quantum adversaries, on which blinded-identity unlinkability depends.
* Whether to allow several blinded identities per address, or to keep the
  one-to-one mapping specified here.
* A fixed starting diversifier index and gap limit for identity recovery.
* How the device obtains assurance that the host's certificate is for the
  $\alpha$ the device derives, and whether the device should display
  $\mathsf{rk}$ during `Register`.
* A CAIP-2 identifier and CAIP-122 namespace for presenting Zcash signing
  identities to services that expect CAIP-10 accounts.
* Completion of the chain bindings: account address derivation, nonce handling,
  the wrapped certificate format, and normative rendering rules per encoding.
* Whether a later version should add a hash-based proof of knowledge of
  $\mathsf{qsk}$ [^zip-2005] to give post-quantum unforgeability.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^zip-0032]: [ZIP 32: Shielded Hierarchical Deterministic Wallets](zip-0032.rst)

[^zip-0229]: [ZIP 229: Version 6 Transaction Format](zip-0229.md)

[^zip-0244]: [ZIP 244: Transaction Identifier Non-Malleability](zip-0244.rst)

[^zip-0258]: [ZIP 258: Deployment of the NU6.3 Network Upgrade](zip-0258.md)

[^zip-0304]: [ZIP 304: Sapling Address Signatures](zip-0304.rst)

[^zip-0312]: [ZIP 312: FROST for Spend Authorization Multisignatures](zip-0312.rst)

[^zip-0316]: [ZIP 316: Unified Addresses and Unified Viewing Keys](zip-0316.rst)

[^zip-2005]: [ZIP 2005: Ironwood Quantum Recoverability](zip-2005.md)

[^zip-2006]: [ZIP 2006: Restricting Transfers into the Orchard Pool](zip-2006.md)

[^RFC3986]: [RFC 3986: Uniform Resource Identifier (URI): Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986)

[^RFC4648]: [RFC 4648: The Base16, Base32, and Base64 Data Encodings](https://www.rfc-editor.org/rfc/rfc4648)

[^RFC7693]: [RFC 7693: The BLAKE2 Cryptographic Hash and Message Authentication Code (MAC)](https://www.rfc-editor.org/rfc/rfc7693)

[^bip-0322]: [BIP 322: Generic Signed Message Format](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki)

[^bip-0350]: [BIP 350: Bech32m format for v1+ witness addresses](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki)

[^eip-152]: [EIP-152: Add BLAKE2 compression function F precompile](https://eips.ethereum.org/EIPS/eip-152)

[^eip-155]: [EIP-155: Simple replay attack protection](https://eips.ethereum.org/EIPS/eip-155)

[^eip-191]: [ERC-191: Signed Data Standard](https://eips.ethereum.org/EIPS/eip-191)

[^eip-712]: [EIP-712: Typed structured data hashing and signing](https://eips.ethereum.org/EIPS/eip-712)

[^eip-1271]: [ERC-1271: Standard Signature Validation Method for Contracts](https://eips.ethereum.org/EIPS/eip-1271)

[^eip-2612]: [ERC-2612: Permit Extension for EIP-20 Signed Approvals](https://eips.ethereum.org/EIPS/eip-2612)

[^eip-4361]: [ERC-4361: Sign-In with Ethereum](https://eips.ethereum.org/EIPS/eip-4361)

[^erc-4337]: [ERC-4337: Account Abstraction Using Alt Mempool](https://eips.ethereum.org/EIPS/eip-4337)

[^eip-7702]: [EIP-7702: Set Code for EOAs](https://eips.ethereum.org/EIPS/eip-7702)

[^erc-7730]: [ERC-7730: Structured Data Clear Signing Format](https://eips.ethereum.org/EIPS/eip-7730)

[^erc-7739]: [ERC-7739: Readable Typed Signatures for Smart Accounts](https://eips.ethereum.org/EIPS/eip-7739)

[^caip-2]: [CAIP-2: Blockchain ID Specification](https://chainagnostic.org/CAIPs/caip-2)

[^caip-122]: [CAIP-122: Sign in With X (SIWx)](https://chainagnostic.org/CAIPs/caip-122)

[^solana-offchain]: [Solana: Off-Chain Message Signing](https://github.com/solana-labs/solana/blob/master/docs/src/proposals/off-chain-message-signing.md)

[^siws]: [Sign In With Solana](https://github.com/phantom/sign-in-with-solana)

[^adr-036]: [Cosmos SDK ADR 036: Arbitrary Message Signature Specification](https://docs.cosmos.network/main/build/architecture/adr-036-arbitrary-signature)

[^adr-050]: [Cosmos SDK ADR 050: SIGN_MODE_TEXTUAL](https://docs.cosmos.network/main/build/architecture/adr-050-sign-mode-textual)
