---
title: Transaction Metadata for EIP-8130
description: A convention for attaching metadata, including a data suffix, to EIP-8130 transactions via a reserved sink address
author: Chris Hunter (@chunter-cb) <chris.hunter@coinbase.com>
discussions-to: https://ethereum-magicians.org/t/erc-transaction-metadata-for-eip-8130
status: Draft
type: Standards Track
category: ERC
created: 2026-06-11
requires: 2028, 8021, 8130
---

## Abstract

[EIP-8130](./eip-8130.md) replaces the single `tx.input` byte string of legacy transactions with a structured `calls` array of execution phases, leaving transaction *metadata* (most commonly a **data suffix**, including [ERC-8021](./eip-8021.md) builder codes) without a home. This proposal reserves a single, codeless **metadata sink address**. A call in the transaction's `calls` whose `to` field equals the sink address is a metadata record: its `data` carries signed, opaque metadata and, because the address has no code (and [EIP-8130](./eip-8130.md) calls carry no value), the call is a guaranteed no-op that a node MAY skip dispatching. The metadata stays in the signed transaction calldata regardless of execution outcome, and indexers read it by filtering `calls` on `to`. To aid routing, the payload SHOULD begin with a 2-byte content descriptor (for example `0x8021` for an [ERC-8021](./eip-8021.md) data suffix); the descriptor is advisory, and unrecognized or absent descriptors are treated as opaque application data.

## Motivation

Wallets and applications attach metadata to transactions for attribution and analytics. On legacy transaction types this metadata is a **data suffix**: extra bytes appended to the end of `tx.input`, a convention that predates [ERC-8021](./eip-8021.md). [ERC-8021](./eip-8021.md) defines a *structure* for such bytes so that parsers can read builder codes consistently. The terms nest: an [ERC-8021](./eip-8021.md) builder code is a data suffix, and a data suffix is metadata.

[EIP-8130](./eip-8130.md) groups calls into batches, which broadens what metadata is useful for beyond a single transaction-level builder code:

- **Builder attribution**: an [ERC-8021](./eip-8021.md) builder code identifying the wallet builder that constructed the transaction.
- **Multi-application batching**: when a wallet bundles calls from several applications into one transaction, per-application attribution lets analytics and revenue be split correctly across the contributors.
- **Payments and remittance**: an invoice number, payment reference, or memo attached to a specific transfer or payout in a batch.
- **Intents and routing**: a tag marking a group of calls as one intent, solver route, or logical action for indexers and portfolio tools.
- **Privacy-preserving metadata**: a commitment to off-chain data, carried as opaque bytes when the data should not be public, a case [ERC-8021](./eip-8021.md) places out of scope.

These differ in *scope*: some describe the whole transaction, some a set of calls, some a single call, which the [EIP-8130](./eip-8130.md) `calls` structure accommodates naturally (see [Wallet behavior](#wallet-behavior)).

[EIP-8130](./eip-8130.md) splits execution into a structured `calls` array (a list of phases, each an ordered, atomic batch of `[to, data]` calls), so the trailing-bytes location no longer exists. This structure is a natural carrier: an [EIP-8130](./eip-8130.md) call is just a `to` and `data` and carries no value at all, so a metadata record is simply a call to a reserved address, with no new top-level transaction field and nothing that can be confused with a value transfer. Because the reserved address is constant, it compresses away in rollup batches, so the marginal L1 cost approximates that of an opaque trailing field while keeping the metadata inside the structured, individually signed transaction body.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Metadata sink address

This proposal reserves a single, codeless 20-byte address as the **metadata sink**:

| Purpose | Sink address |
| --- | --- |
| Transaction metadata | `0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada7a` |

A **metadata call** is a call within an [EIP-8130](./eip-8130.md) transaction's `calls` (in any phase) whose `to` field equals the sink address. Its `data` field is opaque metadata: arbitrary bytes whose interpretation is left to the producing and consuming applications.

### Content descriptor

To let consumers route a payload without prior knowledge of its producer, a metadata call's `data` SHOULD begin with a 2-byte **content descriptor** identifying the payload format:

```
data = descriptor (2 bytes) || payload
```

This document defines a single descriptor:

| Descriptor | Payload | Meaning |
| --- | --- | --- |
| `0x8021` | `schemaId \|\| schemaData`, the [ERC-8021](./eip-8021.md) schema data parsed forwards with the `ercMarker` omitted | The payload is [ERC-8021](./eip-8021.md) attribution. |

Under `0x8021` the sink address and descriptor already identify the payload as [ERC-8021](./eip-8021.md), so the descriptor takes the role of [ERC-8021](./eip-8021.md)'s 16-byte `ercMarker`: the payload is the [ERC-8021](./eip-8021.md) schema data parsed *forwards* (`schemaId` then `schemaData`) with the `ercMarker` omitted, and because the call's `data` is length-delimited the trailing variable-length field MAY run to the end of `data`. See [ERC-8021](./eip-8021.md)'s EIP-8130 integration for this framed encoding.

Future ERCs MAY define additional descriptors. The descriptor is **advisory**: a consumer that does not recognize the leading bytes, that finds no descriptor, or whose payload does not validate as the named format MUST treat the entire `data` as opaque application-defined metadata rather than failing. Because the descriptor is only a hint, a defined format SHOULD be self-validating (structurally checkable) so that a coincidental match on opaque bytes is rejected; a consumer MUST confirm the payload parses as the named format before relying on it. Applications using a private payload format SHOULD either omit the descriptor (treating `data` as wholly opaque) or use a defined descriptor, to avoid colliding with a defined format.

[ERC-8021](./eip-8021.md) (`0x8021`) SHOULD be used when the metadata is **attribution that must be interoperable and machine-resolvable**: specifically when attributing the transaction to registered entities (application, wallet, service) via shared codes, when reward or payout routing is required (codes resolve to payout addresses through a Code Registry), or when structured multi-entity attribution is needed. Opaque metadata SHOULD be used for application-private or freeform data that needs no shared registry, such as a per-transfer memo.

### Scope

Only calls that appear directly in the transaction's `calls` (the signed phases) are metadata. A call to the sink address that originates from within EVM execution (an internal call or subcall made while a call is dispatched) is not a metadata record and MUST NOT be indexed as one. This proposal does not change how metadata is parsed on any non-[EIP-8130](./eip-8130.md) transaction type.

### Metadata call semantics

For every metadata call:

- The `data` field is the metadata payload, optionally prefixed by a content descriptor as described in [Content descriptor](#content-descriptor). [EIP-8130](./eip-8130.md) calls carry no value, so a metadata call is inherently value-less.
- A metadata call has no execution side effects. The sink address is codeless (see [Security Considerations](#security-considerations)), so dispatching the call under standard [EIP-8130](./eip-8130.md) semantics is an immediate no-op: no code runs, no storage is touched, no logs are emitted, and the call cannot revert into or alter any other call.

Metadata payload bytes are part of the transaction, covered by the sender signature (and the payer signature, when present) under the [EIP-8130](./eip-8130.md) authentication model, and charged at the [EIP-2028](./eip-2028.md) per-byte calldata rates like any other bytes in `calls`.

### Dispatch and gas

Because the sink address is codeless, a metadata call is a guaranteed no-op, so a node MAY choose not to dispatch it (recording the `to` and `data` without creating a call frame) to save the work and gas of dispatching to an empty account. Dispatched or not, the metadata bytes remain in the signed transaction and the observable state is identical.

A metadata call SHOULD NOT affect execution and, because a node MAY skip dispatch, SHOULD NOT push an otherwise-valid transaction into out-of-gas. The metadata remains indexable from the signed `calls` regardless of whether the transaction's execution phases succeed, revert, or are skipped. Turning the skipped dispatch into a reduced charge to the sender (rather than a local node optimization) requires the protocol-level treatment in [Convention versus protocol-level support](#convention-versus-protocol-level-support).

### Wallet behavior

A data suffix that describes the transaction as a whole (most importantly an [ERC-8021](./eip-8021.md) builder code) SHOULD be placed in its own dedicated phase containing only that single metadata call, appended as the last phase of `calls`. Isolating it in its own phase keeps it out of the atomic execution phases and the work they do: it cannot cause an execution phase to revert, and if an earlier phase reverts (skipping later phases) the metadata is simply never dispatched while remaining present in the signed `calls` for indexing. A transaction MUST contain at most one [ERC-8021](./eip-8021.md) builder code metadata call; a transaction has a single builder.

Metadata that scopes a **set of calls** (for example per-application attribution when one transaction batches calls from several applications) SHOULD be placed as a metadata call at the start of the phase containing that set; consumers associate it with the calls in the same phase. Metadata that scopes a **single call**, such as a per-transfer memo, MAY be placed as a metadata call adjacent to the call it annotates. In both cases call ordering MAY be significant to consumers, so wallets SHOULD order metadata calls to preserve the intended association, and consumers SHOULD treat association as a convention rather than a protocol guarantee.

This document defines placement and association as RECOMMENDED conventions; it does not mandate a single association rule, since applications may require coarser or finer scoping than phase or adjacency.

### Indexer behavior

For every [EIP-8130](./eip-8130.md) transaction, an indexer enumerates `calls` phase by phase, in order, and for each call:

1. If `call.to` equals the sink address, records `call.data` as metadata. The indexer MAY inspect the leading 2 bytes as a content descriptor: for `0x8021`, it attempts to parse the remaining bytes as [ERC-8021](./eip-8021.md) schema data (forwards, `ercMarker` omitted) and, if they validate, extracts codes per [ERC-8021](./eip-8021.md). If the descriptor is unrecognized or absent, or the payload does not validate as the named format, the indexer records the raw `data` as opaque metadata rather than failing.
2. Otherwise treats the call as an execution call and processes it normally.

Indexers process multiple metadata calls as an ordered list and SHOULD preserve each one's position relative to the surrounding execution calls, so a positional payload (such as a per-transfer memo) can be associated with the adjacent call. Metadata MUST be read from the signed `calls` regardless of per-phase execution status, since a metadata call may be skipped (for example, in a trailing phase after an earlier revert) yet still validly committed by the signer.

## Rationale

### A single sink with an advisory descriptor

A single sink address keeps the scheme minimal: one address to reserve and recognize, and `data` that is simply opaque signed bytes. Typing lives in an optional 2-byte content descriptor at the front of the payload rather than in the address or a mandatory envelope, because the payloads worth detecting are already largely self-validating (an [ERC-8021](./eip-8021.md) payload can be checked structurally), so deterministic typing does not need to be baked into the transaction structure and a coincidental descriptor match on opaque bytes is rejected when the payload fails to validate. Keeping the descriptor advisory (a SHOULD, with unrecognized or absent descriptors treated as opaque) makes the standard permissive: a producer can attach freeform bytes today and a descriptor can be added later without breaking existing consumers.

A per-kind family of addresses, and a mandatory leading type byte, were both considered. Each adds structure (a reserved address per kind, or a required envelope on every payload) that buys little over an advisory descriptor given self-describing payloads, and the address family in particular spends address space and parsing rules on determinism that consumers rarely need.

### Reusing the `calls` array instead of a top-level field

A top-level `dataSuffix` field was considered. It keeps the familiar name but widens the transaction type, and after rollup compression costs effectively the same as a constant sink address. The [EIP-8130](./eip-8130.md) `calls` array is already the right shape: entries are ordered, individually addressed, carry their own `data`, and need no value, so a metadata record reuses the existing structure rather than adding a field. The call format also allows granular metadata per call: in a batch transaction each call can carry or be annotated by its own metadata record, which a single transaction-level suffix or top-level field cannot express.

### Relationship to ERC-8021

[ERC-8021](./eip-8021.md) defines a *payload format* for attribution: entity codes, code registries, and payout routing. This proposal defines the *transport and scope* for metadata on [EIP-8130](./eip-8130.md) transactions. They compose: an [ERC-8021](./eip-8021.md) data suffix is carried as the payload of a metadata call under descriptor `0x8021`. This proposal additionally covers cases [ERC-8021](./eip-8021.md) does not: per-call and per-phase scope ([ERC-8021](./eip-8021.md) is a single transaction-level suffix), signature binding (the metadata is part of the signed `calls` rather than mutable trailing bytes), arbitrary non-attribution metadata, and room for privacy-preserving formats.

### Extensible descriptor space

The 2-byte descriptor leaves ample room for future ERCs to define new payload formats without reserving new addresses or renegotiating an encoding. Because descriptors are advisory and unknown ones are treated as opaque, new formats are forward-compatible with existing indexers.

### Convention versus protocol-level support

As specified, this is an application-layer convention: a metadata call is an ordinary [EIP-8130](./eip-8130.md) call to a reserved address that dispatches to an empty account as a no-op, so the sender still pays the small dispatch cost on top of the [EIP-2028](./eip-2028.md) calldata cost. A follow-on **EIP** could make clients recognize the sink address and skip dispatch entirely (charging only for calldata) and optionally emit a log per metadata call for cheaper, event-based indexing. The on-the-wire shape of a metadata call is identical under both, so adoption can begin as a convention and tighten into protocol behavior later.

### Vanity address

The reserved address uses a mnemonic byte pattern (`0xda7a…` for "data"). It carries no meaning to clients, which treat the sink like any other codeless account.

## Backwards Compatibility

This proposal applies only to [EIP-8130](./eip-8130.md) transactions and changes nothing on legacy transaction types. During the transition, indexers SHOULD continue to parse trailing-bytes data suffixes (including [ERC-8021](./eip-8021.md) builder codes) on legacy transactions while parsing metadata calls on [EIP-8130](./eip-8130.md) transactions. On [EIP-8130](./eip-8130.md) the `0x8021` descriptor and the sink address replace [ERC-8021](./eip-8021.md)'s 16-byte `ercMarker`, so the carried payload is the [ERC-8021](./eip-8021.md) schema data (`schemaId || schemaData`) parsed forwards rather than the legacy reverse-parsed suffix; the schema semantics are otherwise unchanged. A wallet constructing an [EIP-8130](./eip-8130.md) transaction MUST emit metadata as a metadata call, since the `calls` array has no trailing-bytes location.

## Security Considerations

### The sink address must remain codeless

A metadata call is a no-op because the sink address has no code. No party can place code at, or control, the sink: there is no known private key for it, and producing a `CREATE` or `CREATE2` deployment whose resulting address equals the full 20-byte sink requires on the order of 2^160 work. Implementers SHOULD nonetheless confirm the sink address is codeless on each target chain before adoption, and the protocol-level enhancement removes the concern entirely by skipping dispatch regardless of code presence. Because [EIP-8130](./eip-8130.md) calls carry no value, even a hypothetical collision transfers nothing to the sink.

### Unverified metadata

Metadata is an attestation by the signer, not a protocol-verified fact: it asserts only that the signer (and payer, if any) committed to those bytes. Consumers MUST NOT treat metadata as authenticated beyond the transaction signer's identity, MUST NOT grant trust or privileges based on its content without independent verification, and MUST sanitize untrusted payload bytes, including the content descriptor, before use.

### Public metadata

All metadata in a metadata call is public, like any calldata; both opaque payloads and [ERC-8021](./eip-8021.md) attribution are readable by anyone. Producers MUST NOT place sensitive data in these payloads. A producer that needs privacy can carry a commitment to off-chain data as opaque bytes; such a commitment should bind a sufficiently long random salt (for example `keccak256(salt || metadata)`) so that low-entropy metadata cannot be recovered by brute force.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
