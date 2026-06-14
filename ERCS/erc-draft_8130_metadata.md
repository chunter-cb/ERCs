---
title: Transaction Metadata for EIP-8130
description: A convention for attaching metadata, including a data suffix, to EIP-8130 transactions via a reserved sink address
author: Chris Hunter (@chunter-cb) <chris.hunter@coinbase.com>
discussions-to: https://ethereum-magicians.org/t/erc-transaction-metadata-for-eip-8130
status: Draft
type: Standards Track
category: ERC
created: 2026-06-11
requires: 2028, 8130
---

## Abstract

[EIP-8130](./eip-8130.md) replaces the single `tx.input` byte string of legacy transactions with a structured `calls` array of execution phases, leaving transaction *metadata* (such as a **data suffix**) without a home. This proposal reserves a single, codeless **metadata sink address**. A call in the transaction's `calls` whose `to` field equals the sink address is a metadata record: its `data` carries signed, opaque metadata and, because the address has no code (and [EIP-8130](./eip-8130.md) calls carry no value), the call is a guaranteed no-op that a node MAY skip dispatching. The metadata stays in the signed transaction calldata regardless of execution outcome, and indexers read it by filtering `calls` on `to`. This proposal defines only the **transport** and the **scope** (which calls a metadata record describes); the interpretation of the metadata bytes is left to other specifications.

## Motivation

Wallets and applications attach metadata to transactions for attribution and analytics. On legacy transaction types this metadata is a **data suffix**: extra bytes appended to the end of `tx.input`. [EIP-8130](./eip-8130.md) splits execution into a structured `calls` array (a list of phases, each an ordered, atomic batch of `[to, data]` calls), so the trailing-bytes location no longer exists.

[EIP-8130](./eip-8130.md) also groups calls into batches, which broadens what metadata is useful for beyond a single transaction-level suffix:

- **Builder and application attribution**: identifying the wallet builder that constructed the transaction, or the applications whose calls it contains.
- **Multi-application batching**: when a wallet bundles calls from several applications into one transaction, per-application metadata lets analytics and revenue be split correctly across the contributors.
- **Payments and remittance**: an invoice number, payment reference, or memo attached to a specific transfer or payout in a batch.
- **Intents and routing**: a tag marking a group of calls as one intent, solver route, or logical action for indexers and portfolio tools.
- **Privacy-preserving metadata**: a commitment to off-chain data, carried as opaque bytes when the data should not be public.

These differ in *scope*: some describe the whole transaction, some a set of calls, which the [EIP-8130](./eip-8130.md) `calls` structure accommodates naturally (see [Placement and association](#placement-and-association)).

The `calls` array is a natural carrier: an [EIP-8130](./eip-8130.md) call is just a `to` and `data` and carries no value at all, so a metadata record is simply a call to a reserved address, with no new top-level transaction field and nothing that can be confused with a value transfer. Because the reserved address is constant, it compresses away in rollup batches, so the marginal L1 cost approximates that of an opaque trailing field while keeping the metadata inside the structured, individually signed transaction body.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Metadata sink address

This proposal reserves a single, codeless 20-byte address as the **metadata sink**:

| Purpose | Sink address |
| --- | --- |
| Transaction metadata | `0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada7a` |

A **metadata call** is a call within an [EIP-8130](./eip-8130.md) transaction's `calls` (in any phase) whose `to` field equals the sink address.

### Metadata payload

A metadata call's `data` field is **opaque to this specification**: arbitrary bytes whose interpretation is defined by the producing and consuming applications, or by a separate specification (which MAY define a self-identifying prefix). [EIP-8130](./eip-8130.md) calls carry no value, so a metadata call is inherently value-less.

Metadata payload bytes are part of the transaction, covered by the sender signature (and the payer signature, when present) under the [EIP-8130](./eip-8130.md) authentication model, and charged at the [EIP-2028](./eip-2028.md) per-byte calldata rates like any other bytes in `calls`.

### Scope

Only calls that appear directly in the transaction's `calls` (the signed phases) are metadata. A call to the sink address that originates from within EVM execution (an internal call or subcall made while a call is dispatched) is not a metadata record and MUST NOT be indexed as one. This proposal does not change how metadata is parsed on any non-[EIP-8130](./eip-8130.md) transaction type.

### Metadata call semantics

A metadata call has no execution side effects. The sink address is codeless (see [Security Considerations](#security-considerations)), so dispatching the call under standard [EIP-8130](./eip-8130.md) semantics is an immediate no-op: no code runs, no storage is touched, no logs are emitted, and the call cannot revert into or alter any other call.

### Dispatch and gas

Because the sink address is codeless, a metadata call is a guaranteed no-op, so a node MAY choose not to dispatch it (recording the `to` and `data` without creating a call frame) to save the work and gas of dispatching to an empty account. Dispatched or not, the metadata bytes remain in the signed transaction and the observable state is identical.

A metadata call SHOULD NOT affect execution and, because a node MAY skip dispatch, SHOULD NOT push an otherwise-valid transaction into out-of-gas. The metadata remains indexable from the signed `calls` regardless of whether the transaction's execution phases succeed, revert, or are skipped. Turning the skipped dispatch into a reduced charge to the sender (rather than a local node optimization) requires the protocol-level treatment in [Convention versus protocol-level support](#convention-versus-protocol-level-support).

### Placement and association

A transaction MAY contain any number of metadata calls. A metadata call's **scope** is determined by the phase (call array) that contains it:

- If the phase also contains execution (non-metadata) calls, the metadata call tags **that set of calls**.
- If the phase contains only metadata calls, the metadata call describes **the whole transaction** (a data suffix).

To tag an individual call, place that call in its own phase together with a metadata call. Whole-transaction metadata SHOULD be placed in a dedicated phase appended as the last phase of `calls`: a trailing metadata-only phase stays out of the atomic execution phases, so it cannot cause an execution phase to revert, and if an earlier phase reverts (skipping later phases) the metadata is never dispatched while remaining in the signed `calls` for indexing.

This phase-scoped association is what makes the transport useful for **batches**: each phase can carry its own metadata, and each producer (application or wallet) appends its own metadata call without merging into a shared field.

Placement and association are RECOMMENDED conventions, not protocol guarantees; consumers SHOULD treat phase association as a convention.

### Indexer behavior

For every [EIP-8130](./eip-8130.md) transaction, an indexer enumerates `calls` phase by phase, in order, and for each call:

1. If `call.to` equals the sink address, records `call.data` as a metadata record and determines its scope per [Placement and association](#placement-and-association): a metadata call in a phase with execution calls describes those calls; a metadata call in a metadata-only phase describes the whole transaction. The payload bytes are opaque to this specification and interpreted per whatever format applies.
2. Otherwise treats the call as an execution call and processes it normally.

Metadata MUST be read from the signed `calls` regardless of per-phase execution status, since a metadata call may be skipped (for example, in a trailing phase after an earlier revert) yet still validly committed by the signer.

## Rationale

### A single, content-agnostic sink

A single sink address keeps the scheme minimal: one address to reserve and recognize, and `data` that is simply opaque signed bytes. Keeping payload interpretation out of this specification lets the same transport carry attribution, memos, intents, or commitments without this document constraining or enumerating formats; a format specification can define its own bytes (and any self-identifying prefix) independently. A per-kind family of reserved addresses was considered, but it spends address space and parsing rules on a distinction that consumers can make from the payload itself.

### Reusing the `calls` array instead of a top-level field

A top-level `dataSuffix` field was considered. It keeps the familiar name but widens the transaction type, and after rollup compression costs effectively the same as a constant sink address. The [EIP-8130](./eip-8130.md) `calls` array is already the right shape: entries are ordered, individually addressed, carry their own `data`, and need no value, so a metadata record reuses the existing structure rather than adding a field. The call format also allows granular metadata per phase: in a batch transaction each phase can carry its own metadata record (and an individual call can be annotated by isolating it in its own phase), which a single transaction-level suffix or top-level field cannot express.

### Convention versus protocol-level support

As specified, this is an application-layer convention: a metadata call is an ordinary [EIP-8130](./eip-8130.md) call to a reserved address that dispatches to an empty account as a no-op, so the sender still pays the small dispatch cost on top of the [EIP-2028](./eip-2028.md) calldata cost. A follow-on **EIP** could make clients recognize the sink address and skip dispatch entirely (charging only for calldata) and optionally emit a log per metadata call for cheaper, event-based indexing. The on-the-wire shape of a metadata call is identical under both, so adoption can begin as a convention and tighten into protocol behavior later.

### Vanity address

The reserved address uses a mnemonic byte pattern (`0xda7a…` for "data"). It carries no meaning to clients, which treat the sink like any other codeless account.

## Backwards Compatibility

This proposal applies only to [EIP-8130](./eip-8130.md) transactions and changes nothing on legacy transaction types. During the transition, indexers SHOULD continue to parse trailing-bytes data suffixes on legacy transactions while reading metadata calls on [EIP-8130](./eip-8130.md) transactions. A wallet constructing an [EIP-8130](./eip-8130.md) transaction MUST emit metadata as a metadata call, since the `calls` array has no trailing-bytes location. The interpretation of the carried bytes is independent of this transport, so a format previously carried as a trailing suffix can be carried unchanged as a metadata call's `data`.

## Security Considerations

### The sink address must remain codeless

A metadata call is a no-op because the sink address has no code. No party can place code at, or control, the sink: there is no known private key for it, and producing a `CREATE` or `CREATE2` deployment whose resulting address equals the full 20-byte sink requires on the order of 2^160 work. Implementers SHOULD nonetheless confirm the sink address is codeless on each target chain before adoption, and the protocol-level enhancement removes the concern entirely by skipping dispatch regardless of code presence. Because [EIP-8130](./eip-8130.md) calls carry no value, even a hypothetical collision transfers nothing to the sink.

### Unverified metadata

Metadata is an attestation by the signer, not a protocol-verified fact: it asserts only that the signer (and payer, if any) committed to those bytes. Consumers MUST NOT treat metadata as authenticated beyond the transaction signer's identity, MUST NOT grant trust or privileges based on its content without independent verification, and MUST sanitize untrusted payload bytes before use.

### Public metadata

All metadata in a metadata call is public, like any calldata, and readable by anyone. Producers MUST NOT place sensitive data in these payloads. A producer that needs privacy can carry a commitment to off-chain data as opaque bytes; such a commitment should bind a sufficiently long random salt (for example `keccak256(salt || metadata)`) so that low-entropy metadata cannot be recovered by brute force.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
