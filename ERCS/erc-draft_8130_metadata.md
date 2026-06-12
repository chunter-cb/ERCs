---
title: Transaction Metadata for EIP-8130
description: A convention for attaching typed metadata, including a data suffix, to EIP-8130 transactions via a family of reserved sink addresses
author: Chris Hunter (@chunter-cb) <chris.hunter@coinbase.com>
discussions-to: https://ethereum-magicians.org/t/erc-transaction-metadata-for-eip-8130
status: Draft
type: Standards Track
category: ERC
created: 2026-06-11
requires: 2028, 8021, 8130
---

## Abstract

[EIP-8130](./eip-8130.md) replaces the single `tx.input` byte string of legacy transactions with a structured `calls` array of execution phases, leaving transaction *metadata* — most commonly a **data suffix**, including [ERC-8021](./eip-8021.md) builder codes — without a home. This proposal reserves a family of codeless **metadata sink addresses** that share a common 19-byte prefix; the last byte of the address is a **metadata type**. A call in the transaction's `calls` whose `to` carries the metadata prefix is a metadata record: its `data` is the payload for the type named by the address, and because the address has no code (and [EIP-8130](./eip-8130.md) calls carry no value) the call is a guaranteed no-op that a node MAY skip dispatching. The metadata stays in the signed transaction calldata regardless of execution outcome, and indexers read it by matching `to`. Because the address suffix types the payload, one scheme carries [ERC-8021](./eip-8021.md) attribution, opaque application metadata such as memos, and future kinds alike, while `data` remains a pure payload.

## Motivation

Wallets and applications attach metadata to transactions for attribution and analytics. On legacy transaction types this metadata is a **data suffix** — extra bytes appended to the end of `tx.input`, a convention that predates [ERC-8021](./eip-8021.md). [ERC-8021](./eip-8021.md) defines a *structure* for such bytes so that parsers can read builder codes consistently. The terms nest: an [ERC-8021](./eip-8021.md) builder code is a data suffix, and a data suffix is metadata.

[EIP-8130](./eip-8130.md) groups calls into batches, which broadens what metadata is useful for beyond a single transaction-level builder code:

- **Builder attribution**: an [ERC-8021](./eip-8021.md) builder code identifying the wallet builder that constructed the transaction.
- **Multi-application batching**: when a wallet bundles calls from several applications into one transaction, per-application attribution lets analytics and revenue be split correctly across the contributors.
- **Payments and remittance**: an invoice number, payment reference, or memo attached to a specific transfer or payout in a batch.
- **Intents and routing**: a tag marking a group of calls as one intent, solver route, or logical action for indexers and portfolio tools.
- **Privacy-preserving metadata**: the type space leaves room for a future type that carries a commitment to off-chain data, addressing a case [ERC-8021](./eip-8021.md) places out of scope.

These differ in *scope* — some describe the whole transaction, some a set of calls, some a single call — which the [EIP-8130](./eip-8130.md) `calls` structure accommodates naturally (see [Wallet behavior](#wallet-behavior)).

[EIP-8130](./eip-8130.md) splits execution into a structured `calls` array — a list of phases, each an ordered, atomic batch of `[to, data]` calls — so the trailing-bytes location no longer exists. This structure is a natural carrier: an [EIP-8130](./eip-8130.md) call is just a `to` and `data` and carries no value at all, so a metadata record is simply a call to a reserved address — no new top-level transaction field, and nothing that can be confused with a value transfer. Because the reserved address is constant, it compresses away in rollup batches, so the marginal L1 cost approximates that of an opaque trailing field while keeping the metadata inside the structured, individually signed transaction body.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Metadata sink addresses

This proposal reserves a family of codeless 20-byte addresses. An address is a **metadata sink** if its first 19 bytes equal the metadata prefix:

```
prefix = 0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada   (19 bytes)
metadata sink address = prefix || metadataType      (metadataType is 1 byte)
```

The final byte of the address is the **metadata type**, which declares how the call's `data` is interpreted:

| `metadataType` | Sink address | Payload | Meaning |
| --- | --- | --- | --- |
| `0x00` | `0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada00` | arbitrary bytes | Opaque, unspecified application-defined metadata (e.g. a memo, invoice reference, or analytics tag). Interpretation is left to the producing and consuming applications. |
| `0x01` | `0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada01` | a complete [ERC-8021](./eip-8021.md) data suffix | Attribution formatted per [ERC-8021](./eip-8021.md); existing [ERC-8021](./eip-8021.md) parsers apply to `data` unchanged. |

Types `0x02`–`0xff` are reserved for future ERCs.

A **metadata call** is a call within an [EIP-8130](./eip-8130.md) transaction's `calls` (in any phase) whose `to` carries the metadata prefix. Its `data` is the payload for the type named by the final address byte; there is no in-band tag or envelope byte — the address alone determines the type, and `data` is the pure payload.

[ERC-8021](./eip-8021.md) (`0x01`) SHOULD be used when the metadata is **attribution that must be interoperable and machine-resolvable** — specifically when attributing the transaction to registered entities (application, wallet, service) via shared codes, when reward/payout routing is required (codes resolve to payout addresses through a Code Registry), or when structured multi-entity attribution is needed. The opaque type (`0x00`) SHOULD be used for application-private or freeform data that needs no shared registry, such as a per-transfer memo.

### Scope

Only calls that appear directly in the transaction's `calls` (the signed phases) are metadata. A call to a sink address that originates from within EVM execution (an internal call or subcall made while a call is dispatched) is not a metadata record and MUST NOT be indexed as one. This proposal does not change how metadata is parsed on any non-[EIP-8130](./eip-8130.md) transaction type.

### Metadata call semantics

For every metadata call:

- The `data` field is the payload for the address's metadata type, as described in [Metadata sink addresses](#metadata-sink-addresses). [EIP-8130](./eip-8130.md) calls carry no value, so a metadata call is inherently value-less.
- A metadata call has no execution side effects. Sink addresses are codeless (see [Security Considerations](#security-considerations)), so dispatching the call under standard [EIP-8130](./eip-8130.md) semantics is an immediate no-op: no code runs, no storage is touched, no logs are emitted, and the call cannot revert into or alter any other call.

Metadata payload bytes are part of the transaction, covered by the sender signature (and the payer signature, when present) under the [EIP-8130](./eip-8130.md) authentication model, and charged at the [EIP-2028](./eip-2028.md) per-byte calldata rates like any other bytes in `calls`.

### Dispatch and gas

Because sink addresses are codeless, a metadata call is a guaranteed no-op, so a node MAY choose not to dispatch it — recording the `to` and `data` without creating a call frame — to save the work and gas of dispatching to an empty account. Dispatched or not, the metadata bytes remain in the signed transaction and the observable state is identical.

A metadata call SHOULD NOT affect execution and, because a node MAY skip dispatch, SHOULD NOT push an otherwise-valid transaction into out-of-gas. The metadata remains indexable from the signed `calls` regardless of whether the transaction's execution phases succeed, revert, or are skipped. Turning the skipped dispatch into a reduced charge to the sender (rather than a local node optimization) requires the protocol-level treatment in [Convention versus protocol-level support](#convention-versus-protocol-level-support).

### Wallet behavior

A data suffix that describes the transaction as a whole — most importantly an [ERC-8021](./eip-8021.md) builder code (a call to the `0x…01` sink) — SHOULD be placed in its own dedicated phase containing only that single metadata call, appended as the last phase of `calls`. Isolating it in its own phase keeps it out of the atomic execution phases and the work they do: it cannot cause an execution phase to revert, and if an earlier phase reverts (skipping later phases) the metadata is simply never dispatched while remaining present in the signed `calls` for indexing. A transaction MUST contain at most one [ERC-8021](./eip-8021.md) builder code metadata call; a transaction has a single builder.

Metadata that scopes a **set of calls** — for example per-application attribution when one transaction batches calls from several applications — SHOULD be placed as a metadata call at the start of the phase containing that set; consumers associate it with the calls in the same phase. Metadata that scopes a **single call**, such as a per-transfer memo, MAY be placed as a metadata call adjacent to the call it annotates. In both cases call ordering MAY be significant to consumers, so wallets SHOULD order metadata calls to preserve the intended association, and consumers SHOULD treat association as a convention rather than a protocol guarantee.

This document defines placement and association as RECOMMENDED conventions; it does not mandate a single association rule, since applications may require coarser or finer scoping than phase or adjacency.

### Indexer behavior

For every [EIP-8130](./eip-8130.md) transaction, an indexer enumerates `calls` phase by phase, in order, and for each call:

1. If `call.to` carries the metadata prefix, reads the final byte as `metadataType` and records `call.data` as metadata of that type: for `0x01`, parses the [ERC-8021](./eip-8021.md) data suffix and extracts codes per [ERC-8021](./eip-8021.md); for `0x00`, records the opaque payload; for an unrecognized type, records the raw payload and otherwise ignores it.
2. Otherwise treats the call as an execution call and processes it normally.

Indexers process multiple metadata calls as an ordered list and SHOULD preserve each one's position relative to the surrounding execution calls, so a positional payload (such as a per-transfer memo) can be associated with the adjacent call. Metadata MUST be read from the signed `calls` regardless of per-phase execution status, since a metadata call may be skipped (for example, in a trailing phase after an earlier revert) yet still validly committed by the signer.

## Rationale

### Typing by address suffix

Encoding the metadata type in the final byte of the address keeps `data` a pure payload and lets indexers classify a call from `to` alone — no envelope byte to strip and no need to read `data` to learn the kind. An [ERC-8021](./eip-8021.md) suffix therefore sits in `data` exactly as it would on legacy calldata, so existing parsers apply with no offset. A leading type byte inside `data` was considered; it works, but it forces every consumer (including unmodified [ERC-8021](./eip-8021.md) parsers) to account for the prefix and to read into `data` before classifying. Unrelated per-kind addresses were also considered; sharing a 19-byte prefix instead makes the whole set recognizable by a single prefix match, compressible as a family of constants, and — because no key or `CREATE2` deployment can realistically match a 19-byte prefix (roughly 2^152 work) — codeless by construction. Each type address is a constant, so it back-references away in rollup batches just like a single sink would.

### Reusing the `calls` array instead of a top-level field

A top-level `dataSuffix` field was considered. It keeps the familiar name but widens the transaction type, and after rollup compression costs effectively the same as a constant sink address. The [EIP-8130](./eip-8130.md) `calls` array is already the right shape: entries are ordered, individually addressed, carry their own `data`, and need no value — so a metadata record reuses the existing structure rather than adding a field.

### Relationship to ERC-8021

[ERC-8021](./eip-8021.md) defines a *payload format* for attribution: entity codes, code registries, and payout routing. This proposal defines the *transport and scope* for metadata on [EIP-8130](./eip-8130.md) transactions. They compose — an [ERC-8021](./eip-8021.md) data suffix is carried in a `0x…01` metadata call — and this proposal additionally covers cases [ERC-8021](./eip-8021.md) does not: per-call and per-phase scope ([ERC-8021](./eip-8021.md) is a single transaction-level suffix), signature binding (the metadata is part of the signed `calls` rather than mutable trailing bytes), arbitrary non-attribution metadata, and room for privacy-preserving types.

### Extensible type space

The 1-byte type leaves 254 unused values for future ERCs to define new metadata kinds — for example a commitment to off-chain data for privacy-preserving metadata — without reserving new address families, renegotiating an encoding, or coordinating a registry. Unknown types are recorded raw and ignored, so new kinds are forward-compatible with existing indexers.

### Convention versus protocol-level support

As specified, this is an application-layer convention: a metadata call is an ordinary [EIP-8130](./eip-8130.md) call to a reserved address that dispatches to an empty account as a no-op, so the sender still pays the small dispatch cost on top of the [EIP-2028](./eip-2028.md) calldata cost. A follow-on **EIP** could make clients recognize the metadata prefix and skip dispatch entirely — charging only for calldata — and optionally emit a log per metadata call for cheaper, event-based indexing. The on-the-wire shape of a metadata call is identical under both, so adoption can begin as a convention and tighten into protocol behavior later.

### Vanity prefix

The reserved prefix uses a mnemonic byte pattern (`0xda7a…` for "data"). It carries no meaning to clients, which treat any address in the family like any other codeless account.

## Backwards Compatibility

This proposal applies only to [EIP-8130](./eip-8130.md) transactions and changes nothing on legacy transaction types. During the transition, indexers SHOULD continue to parse trailing-bytes data suffixes (including [ERC-8021](./eip-8021.md) builder codes) on legacy transactions while parsing metadata calls on [EIP-8130](./eip-8130.md) transactions. The [ERC-8021](./eip-8021.md) payload format is unchanged — the same bytes, relocated from `tx.input` to the `data` of a `0x…01` metadata call — so existing [ERC-8021](./eip-8021.md) parsers apply directly to `call.data`. A wallet constructing an [EIP-8130](./eip-8130.md) transaction MUST emit metadata as a metadata call, since the `calls` array has no trailing-bytes location.

## Security Considerations

### Sink addresses cannot hold code

A metadata call is a no-op because the sink address has no code. The 19-byte shared prefix makes this structural: producing a private key, or a `CREATE`/`CREATE2` deployment, whose resulting address matches a 19-byte prefix requires on the order of 2^152 work, so no party can deploy code at, or control, any address in the metadata family. Implementers SHOULD nonetheless confirm the specific sink addresses in use are codeless on each target chain before adoption, and the protocol-level enhancement removes the concern entirely by skipping dispatch regardless of code presence. Because [EIP-8130](./eip-8130.md) calls carry no value, even a hypothetical collision transfers nothing to a sink.

### Unverified metadata

Metadata is an attestation by the signer, not a protocol-verified fact: it asserts only that the signer (and payer, if any) committed to those bytes. Consumers MUST NOT treat metadata as authenticated beyond the transaction signer's identity, MUST NOT grant trust or privileges based on its content without independent verification, and MUST sanitize untrusted payload bytes before use.

### Public metadata

All metadata in a metadata call is public, like any calldata; the defined types (`0x00`, `0x01`) are readable by anyone. Producers MUST NOT place sensitive data in these payloads. A future privacy-preserving type that carries an off-chain commitment should bind a sufficiently long random salt (for example `keccak256(salt || metadata)`) so that low-entropy metadata cannot be recovered by brute force.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
