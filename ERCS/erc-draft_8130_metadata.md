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

[EIP-8130](./eip-8130.md) replaces the single `tx.input` byte string of legacy transactions with a structured `calls` array of execution phases, leaving transaction *metadata* (such as a **data suffix**) without a home. This proposal reserves a single, codeless **metadata sink address**. A call whose `to` equals the sink is a metadata record: its `data` carries opaque, signed bytes and is a guaranteed no-op that a node MAY skip dispatching. The metadata remains in the signed transaction regardless of execution outcome and is recovered by indexers by filtering `calls` on `to`. This proposal defines only the **transport** and the **scope** (which calls a metadata record describes); interpretation of the bytes is left to other specifications.

## Motivation

Wallets and applications attach metadata to transactions for attribution and analytics. On legacy transaction types this metadata is a **data suffix**: extra bytes appended to `tx.input`. [EIP-8130](./eip-8130.md) splits execution into a structured `calls` array (a list of phases, each an ordered, atomic batch of `[to, data]` calls), so the trailing-bytes location no longer exists.

Batching also broadens what metadata is useful for beyond a single transaction-level suffix:

- **Builder and application attribution**: identifying the wallet builder or the applications whose calls the transaction contains.
- **Multi-application batching**: per-application metadata lets analytics and revenue be split correctly across contributors when several applications share one transaction.
- **Payments and remittance**: an invoice number, reference, or memo attached to a specific transfer in a batch.
- **Intents and routing**: a tag marking a group of calls as one intent or solver route for indexers and portfolio tools.
- **Privacy-preserving metadata**: a commitment to off-chain data, carried as opaque bytes.

These differ in *scope* (whole transaction vs. a set of calls), which the `calls` structure accommodates naturally (see [Placement and association](#placement-and-association)). Because the sink address is constant it compresses away in rollup batches, so the marginal L1 cost approximates that of an opaque trailing field while keeping metadata inside the structured, signed transaction body.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Metadata sink address

This proposal reserves a single, codeless 20-byte address as the **metadata sink**:

| Purpose | Sink address |
| --- | --- |
| Transaction metadata | `0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada7a` |

A **metadata call** is a call within an [EIP-8130](./eip-8130.md) transaction's `calls` (in any phase) whose `to` equals the sink address. Because the sink is codeless and [EIP-8130](./eip-8130.md) calls carry no value, a metadata call is a guaranteed no-op: no code runs, no storage is touched, no logs are emitted, and the call cannot revert into or affect any other call. A node MAY skip dispatching it entirely (recording `to` and `data` without creating a call frame); either way, the bytes remain in the signed transaction and observable state is identical. A metadata call SHOULD NOT push an otherwise-valid transaction into out-of-gas, and its `data` is indexable regardless of whether surrounding execution phases succeed or revert. The on-the-wire shape of a metadata call is identical whether or not the node skips dispatch, so indexers have the same guarantees as for any other call in `calls`.

### Metadata payload

A metadata call's `data` is **opaque to this specification**: arbitrary bytes interpreted by the producing and consuming applications, or by a separate specification (which MAY define a self-identifying prefix). Payload bytes are part of the transaction, covered by the sender signature (and payer signature, when present) under the [EIP-8130](./eip-8130.md) authentication model, and charged at [EIP-2028](./eip-2028.md) per-byte calldata rates.

### Scope

Only calls appearing directly in the transaction's `calls` (the signed phases) are metadata records. A call to the sink address originating from within EVM execution (an internal subcall) is not a metadata record and MUST NOT be indexed as one. This proposal does not change how metadata is parsed on any non-[EIP-8130](./eip-8130.md) transaction type.

### Placement and association

A transaction MAY contain any number of metadata calls. A metadata call's **scope** is determined by the phase (call array) that contains it:

- If the phase also contains execution (non-metadata) calls, the metadata call tags **that set of calls**.
- If the phase contains only metadata calls, it describes **the whole transaction** (a data suffix).

To tag an individual call, place it in its own phase together with a metadata call. Whole-transaction metadata SHOULD be appended as the last phase of `calls`: a trailing metadata-only phase stays out of the atomic execution phases, so it cannot cause an earlier phase to revert, and if an earlier phase reverts (skipping later phases) the metadata is never dispatched while remaining in the signed `calls` for indexing.

This phase-scoped model is what makes the transport useful for batches: each phase can carry its own metadata, and each producer (application or wallet) appends its own metadata call without merging into a shared field.

Placement and association are RECOMMENDED conventions, not protocol guarantees; consumers SHOULD treat phase association as a convention.

### Indexer behavior

For every [EIP-8130](./eip-8130.md) transaction, an indexer enumerates `calls` phase by phase, in order:

1. If `call.to` equals the sink address, record `call.data` as a metadata record. Determine its scope per [Placement and association](#placement-and-association): mixed phase means that phase's execution calls; metadata-only phase means the whole transaction.
2. Otherwise, treat the call as an execution call.

Metadata MUST be read from the signed `calls` regardless of per-phase execution status, since a skipped metadata call (for example in a trailing phase after an earlier revert) was still validly committed by the signer.

## Rationale

### A single, content-agnostic sink

A single sink address requires only one reserved address and leaves `data` as plain opaque bytes. Keeping payload interpretation out of this specification lets the same transport carry attribution, memos, intents, or commitments without this document enumerating formats; a format can define its own bytes (including any self-identifying prefix) independently. A per-kind family of reserved addresses was considered, but it spends address space and parsing rules on a distinction that consumers can make from the payload itself.

### Reusing the `calls` array instead of a top-level field

A top-level `dataSuffix` field was considered. It keeps a familiar name but widens the transaction type and, after rollup compression, costs the same as a constant sink address. The `calls` array is already the right shape: entries are ordered, individually addressed, carry their own `data`, and need no value, so a metadata record reuses the existing structure. Using phases also allows granular metadata per set of calls, which a single top-level field cannot express.

### Vanity address

The reserved address uses a mnemonic byte pattern (`0xda7a…` for "data"). It carries no meaning to clients, which treat the sink like any other codeless account.

## Backwards Compatibility

This proposal applies only to [EIP-8130](./eip-8130.md) transactions and changes nothing on legacy types. During transition, indexers SHOULD continue to parse trailing-bytes data suffixes on legacy transactions while reading metadata calls on [EIP-8130](./eip-8130.md) transactions. A wallet constructing an [EIP-8130](./eip-8130.md) transaction MUST emit metadata as a metadata call; the `calls` array has no trailing-bytes location. Payload interpretation is independent of this transport, so a format previously carried as a trailing suffix can be carried unchanged as a metadata call's `data`.

## Security Considerations

### Unverified metadata

Metadata is an attestation by the signer only: it asserts that the signer committed to those bytes, not that the content is true. Consumers MUST NOT grant trust or privileges based on payload content without independent verification, and MUST sanitize untrusted bytes before use.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
