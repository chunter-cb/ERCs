---
eip: TBD
title: Account-Scoped Typed Signature Verification
description: Two signature profiles over the EIP-8130 Keystore, a domain-constructing onchain verifier for typed data and an account-bound personal-sign profile for offchain authentication.
author: Chris Hunter (@chunter-cb)
discussions-to: TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-08-05
requires: 191, 712, 8130
---

## Abstract

This ERC defines two signature profiles over one verification engine, the [EIP-8130](./eip-8130.md) Keystore, covering every signature flow an account produces.

**Profile A (consumer-bound, typed, onchain)** defines `TypedSigVerifier`, an immutable singleton deployed at a canonical address on every chain, which verifies [EIP-712](./eip-712.md) signatures against an account's Keystore configuration. The verifier constructs the EIP-712 domain itself from values it reads onchain (`chainId` from the chain, `verifyingContract` from `msg.sender`, and the signing account as the domain `salt`), so a consuming contract cannot express an incorrect replay binding.

**Profile B (account-bound, personal-sign, offchain)** defines the digest for free-text message signing (Sign-In with Ethereum and similar flows): the [EIP-191](./eip-191.md) message hash wrapped in an [ERC-7739](./eip-7739.md) `PersonalSign` struct under a domain bound to the signing account. Offchain services verify with a single `eth_call` to the Keystore.

Both profiles return the resolved actor identity and scope rather than a bare validity bit, allowing the consumer (an onchain contract or an offchain service) to make its own authorization decision. Together they can back an existing [ERC-1271](./eip-1271.md) surface or be used directly by new integrations: no call into account code, no per-wallet envelope, and signed content renders natively in wallets.

## Motivation

EIP-712 solved digest binding in 2017: a correctly constructed domain commits to the chain and the verifying contract, and [ERC-2612](./eip-2612.md) shows the full discipline. The failure mode in practice has never been the hash construction; it has been that nothing on the verification path *enforces* it. `ecrecover` and ERC-1271 both accept arbitrary 32-byte digests, so binding correctness is re-implemented independently by every integrator, and drifts: omitted chain identifiers, domain separators cached across forks, nonconforming permit variants.

The cost of that drift has risen sharply. CREATE2 counterfactual deployment, [EIP-7702](./eip-7702.md), and delegated agent keys mean the same signer material now controls same-address accounts across many chains and many accounts on one chain. An unbound signature is a cross-chain, cross-account skeleton key. Wallet-side mitigations (per-wallet ERC-1271 envelopes, [ERC-7739](./eip-7739.md)) restore binding at the cost of nesting, opaque rendering, and per-wallet integration.

EIP-8130 additionally makes the *result* of verification richer than a boolean: an account has many actors with distinct scopes, and a consumer needs to know *who* signed and *what they may do*, not merely that some key validated. ERC-1271's interface cannot carry that answer.

This ERC therefore moves domain construction into the verifier. The binding stops being a convention every integrator must follow and becomes an invariant no integrator can violate.

### Comparison

| Property | `ecrecover` / ERC-2612 | ERC-1271 | ERC-1271 + ERC-7739 | This ERC |
| --- | --- | --- | --- | --- |
| Signer types | secp256k1 EOA key only | Any (account-defined) | Any (account-defined) | Any EIP-8130 authenticator (k1, P-256, WebAuthn, delegate, …) |
| Replay binding (chain / consumer / account) | By convention; integrator-built digest | None; digest is opaque to the account | Enforced, via account-side envelope | Enforced, verifier-built from onchain facts |
| Verification result | Recovered address | Boolean magic value | Boolean magic value | `(actorId, scope)` |
| Delegate / scoped keys | No | Opaque to the caller | Opaque to the caller | First-class; consumer gates on scope |
| Revocation / expiry | No | Account-defined | Account-defined | Enforced at verification time by the Keystore |
| Call into account code | No | Yes (reentrancy / gas surface) | Yes | No |
| Wallet rendering of signed content | Native EIP-712 | App-dependent; often opaque hash | Readable, but nested per ERC-7739 | Native EIP-712, no envelope |
| Integration shape | Per-token reimplementation | Per-app digest + per-wallet envelope | Per-wallet ERC-7739 support | One call to one canonical address |

## Specification

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Signature profiles

Every signature is classified by its content shape (structured vs free text) and its verifier location (onchain vs offchain). This ERC defines one profile per supported cell:

| Content | Verifier | Profile | Binding target |
| --- | --- | --- | --- |
| EIP-712 typed data | Onchain contract | **A**: TypedSigVerifier | Consumer (`verifyingContract = msg.sender`), account as `salt` |
| EIP-712 typed data | Offchain service | **A** (via `eth_call`) | Same digest; service impersonates no sender, so it names the consumer explicitly |
| Personal-sign text | Offchain service | **B**: account-bound PersonalSign | Account (`verifyingContract = account`) |
| Personal-sign text | Onchain contract | **Deprecated** | N/A |

The binding target deliberately differs: an onchain consumer is a real address that can be read from `msg.sender`, so signatures bind to it; an offchain verifier has no forgery-resistant identity, so signatures bind to the account, and service-level replay is handled in the message content (as in Sign-In with Ethereum's domain, URI, and nonce fields). These are the only two constructions; wallets and integrators MUST NOT introduce additional envelopes.

Raw `eth_sign` (signing an unprefixed digest) and onchain consumption of personal-sign messages are deprecated by this ERC and MUST NOT be supported in new integrations; a contract that needs to consume a signed message MUST define an EIP-712 type and use Profile A.

### Profile A: consumer-bound typed signatures

#### Verifier interface

```solidity
interface ITypedSigVerifier {
    /// @notice Verifies an EIP-712 signature over `structHash` under a domain constructed by this
    ///         contract, resolving it to a live EIP-8130 actor of `account`.
    /// @dev Reverts on any authentication failure (bubbled from Keystore.authenticateActor).
    /// @param account     The EIP-8130 account the signature is resolved against.
    /// @param nameHash    keccak256 of the consumer's domain name.
    /// @param versionHash keccak256 of the consumer's domain version.
    /// @param structHash  EIP-712 hashStruct of the consumer-defined message.
    /// @param auth        Keystore auth blob: authenticator(20) || authenticator-specific data.
    /// @return actorId Identifier of the verified actor.
    /// @return scope   The actor's scope bitmask (0x00 = unrestricted admin).
    function authenticate(
        address account,
        bytes32 nameHash,
        bytes32 versionHash,
        bytes32 structHash,
        bytes calldata auth
    ) external view returns (bytes32 actorId, uint16 scope);
}
```

#### Domain construction

The verifier MUST compute the digest as follows and MUST NOT accept a caller-supplied digest or domain separator:

```solidity
bytes32 constant DOMAIN_TYPEHASH = keccak256(
    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract,bytes32 salt)"
);

domainSeparator = keccak256(abi.encode(
    DOMAIN_TYPEHASH,
    nameHash,
    versionHash,
    block.chainid,                       // read onchain; not caller-supplied
    msg.sender,                          // the consuming contract; not caller-supplied
    bytes32(uint256(uint160(account)))   // the signing account as salt
));

digest = keccak256(abi.encodePacked(hex"1901", domainSeparator, structHash));
```

The verifier MUST then call `Keystore.authenticateActor(account, digest, auth)` on the canonical EIP-8130 Keystore and return its `(actorId, scope)` result unmodified, allowing any revert to bubble. Policy gating is determined from `scope` alone (the `POLICY` bit); a consumer that needs the actor's policy manager reads it from the Keystore as a separate execution-time query.

This yields three unforgeable replay bindings: `chainId` (cross-chain), `verifyingContract` (cross-consumer), and `salt` (cross-account). Consumers MUST NOT additionally encode the account in their struct; the salt is the sole account binding.

#### Consumer requirements

- Consumers MUST treat any revert as verification failure and MUST NOT interpret any default, empty, or zero value as authorization.
- Consumers MUST gate the action on `scope`. For blanket-authority actions (e.g. token approvals), consumers SHOULD require the operational predicate defined by EIP-8130's Scopes vocabulary: `scope == 0`, or `SENDER` set with `POLICY` unset. A policy-gated actor (`POLICY` set) MUST NOT be granted blanket authority on the basis of a signature alone, as its authority is a function of execution context the consumer cannot evaluate.
- Replay protection beyond the domain (nonces) is the consumer's responsibility. Consumers supporting concurrent signers SHOULD use unordered nonces keyed by `(account, actorId)`, with `actorId` taken from the verifier's return value and never from caller input.
- Liveness is evaluated at verification time: revoked and expired actors fail authentication. Consumer-level deadlines are a UX bound, not the security bound.

#### Wallet requirements

Wallets MUST sign the digest as standard EIP-712 typed data with the five-field domain above. There is no envelope, no nested struct, and no wallet-specific wrapping; the consumer's struct renders directly. Wallets SHOULD display the domain `salt` as the signing account when it differs from the connected account.

### Profile B: account-bound personal-sign (Sign-In with Ethereum)

Free-text message signing ([ERC-4361](./eip-4361.md) Sign-In with Ethereum and equivalent offchain authentication flows) uses the account-bound PersonalSign construction defined by EIP-8130's reference `AccountDomain` library and matches the account's own ERC-1271 wrap bit-for-bit:

```solidity
bytes32 constant PROFILE_B_DOMAIN_TYPEHASH = keccak256(
    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
);
bytes32 constant PERSONAL_SIGN_TYPEHASH = keccak256("PersonalSign(bytes prefixed)");
bytes32 constant NAME_HASH = keccak256("EIP8130Account");
bytes32 constant VERSION_HASH = keccak256("1");

// message: the raw text the user sees (e.g. the SIWE message)
bytes32 eip191Hash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n", len(message), message));
bytes32 structHash = keccak256(abi.encode(PERSONAL_SIGN_TYPEHASH, eip191Hash));

domainSeparator = keccak256(abi.encode(
    PROFILE_B_DOMAIN_TYPEHASH, NAME_HASH, VERSION_HASH,
    chainId,      // the chain named in the message (e.g. SIWE Chain ID)
    account       // the signing account: verifyingContract = account
));

digest = keccak256(abi.encodePacked(hex"1901", domainSeparator, structHash));
```

#### Verification

An offchain service verifies by computing `digest` locally and issuing a single `eth_call` to `Keystore.authenticateActor(account, digest, auth)` on the chain named in the message, treating any revert as failure and receiving `(actorId, scope)` on success. No account code is executed for verification; counterfactual accounts with an inline k1 self (every EOA) verify with no prior deployment.

The account binding (`verifyingContract = account`) makes a signature invalid for any other account sharing the same key. Cross-*service* replay is not bound in the domain (there is no forgery-resistant service identity offchain) and MUST instead be enforced in the message content, which ERC-4361's `domain`, `uri`, `nonce`, `issued-at`, and `expiration-time` fields already require; services MUST validate those fields as ERC-4361 specifies.

#### Sign-in authorization

Services MUST gate sign-in on `scope`, and the default MUST be the operational predicate (`scope == 0`, or `SENDER` set with `POLICY` unset): signing in *as the account* is blanket impersonation, and a policy-gated actor's authority is contextual in a way an authentication flow cannot evaluate.

Because the service receives `actorId`, it MAY additionally bind the resulting session to the specific key that authenticated rather than to the account alone: distinct session policies for an admin key versus a delegate, per-key audit trails, and per-key session revocation. This is the sanctioned pattern for letting non-operational actors authenticate: a service MAY accept a policy-gated actor's signature for an *actorId-scoped* session (the agent authenticating as itself, with service-side privileges to match), while account-level sessions remain operational-only.

Future EIP-8130 scope bits specific to sign-in (e.g. a grant permitting authentication but not execution, or excluding an otherwise-operational key from sign-in) MAY be defined in the Scopes vocabulary; its append-only assignment makes this safe against deployed configurations, and services following the rules above inherit correct behavior for such bits without changes.

Because verification reads live Keystore state, revoking or expiring an actor invalidates its sign-in capability immediately. Sessions already issued from a prior sign-in are outside this ERC's scope and remain governed by the service's session policy; services SHOULD bound session lifetimes accordingly.

#### Counterfactual smart accounts

A counterfactual smart account (CREATE2-predicted, not yet deployed) has no Keystore actor state, so Profile B verification fails until first deployment. EOAs are unaffected; the inline k1 self exists implicitly. An [ERC-6492](./eip-6492.md)-style wrapper for pre-deployment verification MAY be specified separately and does not modify this profile.

### Relationship to raw `authenticateActor`

`Keystore.authenticateActor(account, hash, auth)` remains public and is normatively reserved for account implementations, offchain verifiers computing a digest defined by this ERC (Profile B), and protocols that own and fully specify their digest construction (e.g. the Keystore's own signed change operations, an account's ERC-1271 wrap). Contracts consuming user signatures MUST use the TypedSigVerifier and MUST NOT call `authenticateActor` with a digest of their own construction. The two addresses thus encode opposite trust postures (digest-owning callers use the Keystore, digest-consuming callers use the verifier), and integration reviews SHOULD treat any direct `authenticateActor` call with a locally constructed digest as requiring justification.

### Deployment

The verifier is immutable, holds no storage, and is deployed via deterministic CREATE2 to the same address on every chain (address TBD upon finalization). Chains implementing EIP-8130 natively MAY execute the verifier natively; a native implementation MUST be observationally equivalent to this specification, including digest construction as if `verifyingContract` were the calling consumer.

### Support matrix

| Environment | Verification path | Notes |
| --- | --- | --- |
| EIP-8130 chain, native consumer (e.g. protocol token) | Native verifier + native Keystore | Fast path; equivalence with the EVM path is normative |
| EIP-8130 chain, EVM consumer | Canonical verifier contract → Keystore | Same address, same digest, same result as native |
| Non-8130 chain | Canonical verifier contract → Keystore | Full functionality; callback-free verification everywhere |
| Plain EOA signer | Any of the above | The EOA is the account's inline k1 actor; no registration needed |
| Offchain service (SIWE, passkey login) | Profile B digest + `eth_call` to Keystore | One RPC call; live revocation; `(actorId, scope)` for key-aware sessions |
| Legacy app (predates this ERC) | Account's ERC-1271 surface | Compatibility floor; admin/operational actors only, no scope result |

## Rationale

**Verifier-constructed domain over caller-constructed digest.** Every field a caller could misstate is instead read onchain. This converts EIP-712's convention into an invariant, which is the entire contribution; a library encoding the same construction would preserve the footgun it exists to remove.

**Salt over struct field for the account.** The account is not the verifying contract, so `verifyingContract` cannot carry it; `salt` is the standard EIP-712 domain member with existing wallet support. Placing it in the domain rather than the consumer's struct makes the binding uniform across all consumers and non-omittable.

**Separate singleton over a Keystore method.** The Keystore is a pure authorization oracle and is deliberately signing-domain-agnostic; signing profiles are the most revision-prone layer of the stack (EIP-191 was repaired by EIP-712, EIP-712-through-ERC-1271 by ERC-7739), while authorization state is the most frozen. Housing the profile in a distinct immutable contract decouples those lifetimes: a profile mistake or a successor signing standard is handled by deploying a new verifier address and migrating wallets, leaving the Keystore and all account state untouched. Merged, the same event would require changing the canonical authorization contract (on natively accelerated chains, the predeploy) across every chain simultaneously. The separation also preserves trust legibility: calling the Keystore asserts "I own and fully specify my digest," calling the verifier asserts "bind it for me." Two addresses make that posture auditable in integration review; one address puts the raw entry point adjacent to the safe one for every integrator.

**Keystore digest construction is not a counterexample.** The Keystore constructs EIP-712 digests for its own signed operations (actor changes, lock operations). Those are self-owned digests, precisely what the raw-entry-point rule prescribes for any digest-owning protocol. What the Keystore never does is construct digests on behalf of third-party consumers; that service is the verifier's entire function and sits on the consumer side of the interface.

**Scoped result over ERC-1271's boolean.** A bare magic value forces the account to decide authorization for actions it cannot see. Returning `(actorId, scope)` moves the decision to the contract that has the context, and gives it the actor identity needed for per-actor accounting such as nonce namespacing. The policy manager is deliberately excluded from the return: it is execution-time state, and a signature consumer that needs it should read it fresh from the Keystore rather than receive a value that may be stale by use time.

**Two profiles, not one.** The correct binding target differs by verifier: an onchain consumer has an unforgeable identity (`msg.sender`) and gets the signature bound to it; an offchain verifier does not, so the signature binds to the account and service-level replay lives in the message content, where ERC-4361 already puts it. Collapsing to one construction would either leave onchain consumers unbound or force offchain flows to name a fictitious contract. Fixing exactly two named constructions (and reusing the account's existing ERC-1271 digest as Profile B verbatim) is what prevents the per-wallet envelope zoo from re-forming.

**Key-aware sign-in over account-boolean sign-in.** Returning `(actorId, scope)` to an authentication service upgrades sign-in from "some key of this account validated" to "this specific key validated, with this authority": passkey login via a WebAuthn actor with no bridging infrastructure, sessions and audit trails scoped to the key that logged in, and immediate lockout on key revocation; none of which ERC-1271's magic value can express.

## Backwards Compatibility

ERC-1271 is unaffected and remains available on EIP-8130 account implementations as a compatibility surface for existing integrations. This ERC introduces no changes to the EIP-8130 Keystore. Signatures produced under this profile are not valid under any account's ERC-1271 envelope and vice versa; the two surfaces do not overlap.

## Security Considerations

- **Fail-closed consumption.** The verifier reverts on failure; consumers wrapping it in `try/catch` MUST map catch to rejection.
- **Signature availability.** A valid signature is authorization-bearing until the actor is revoked or expired. Consumers SHOULD enforce deadlines and MUST enforce nonces; signers SHOULD scope and expire keys used with signature-consuming protocols.
- **Cross-consumer reuse is impossible by construction** (`verifyingContract = msg.sender`), including by a malicious consumer replaying a signature gathered by another: the replay changes `msg.sender` and the digest no longer matches.
- **Policy actors.** Accepting a policy-gated actor's signature for blanket authority bypasses its policy gate; the scope requirements above are load-bearing and consumers MUST NOT relax them. For sign-in this means account-level sessions are operational-only; a policy-gated actor's sign-in, where accepted, MUST be confined to an actorId-scoped session.
- **Profile confusion.** The two profiles produce disjoint digests for any input (different domain typehashes and verifyingContract semantics), so a signature made under one cannot validate under the other. Verifiers MUST NOT accept both constructions for the same flow.
- **Offchain verification trust.** A Profile B verifier trusts its RPC endpoint for Keystore state; services with strong requirements SHOULD verify against multiple providers or a light client. Session issuance after a successful sign-in is a service-side decision this ERC does not bound; revoking an actor does not revoke sessions it already opened.
- **Native equivalence.** Divergence between a native implementation and this specification would make a signature valid on one implementation and invalid on another for the same logical state. Differential testing against the reference implementation is REQUIRED for native deployments, with digest construction as the primary target.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
