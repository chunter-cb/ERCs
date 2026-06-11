---
title: Transaction Metadata Sinks for EIP-8130
description: Reserved sink addresses that let EIP-8130 calls carry signed, non-executing attribution metadata such as ERC-8021 builder codes and data suffixes
author: Chris Hunter (@chunter-cb) <chris.hunter@coinbase.com>
discussions-to: https://ethereum-magicians.org/t/erc-transaction-metadata-sinks-for-eip-8130
status: Draft
type: Standards Track
category: ERC
created: 2026-06-11
requires: 2028, 8021, 8130
---

## Abstract

[EIP-8130](./eip-8130.md) replaces the single `tx.input` byte string of legacy transactions with a structured `calls` array. Attribution metadata that previously rode as trailing bytes on `tx.input` (the "data suffix" convention, including [ERC-8021](./eip-8021.md) builder codes) no longer has a natural home. This proposal reserves well-known **metadata sink addresses**. A call whose `to` field equals a sink address is a metadata record: it carries signed bytes in `call.data`, performs no dispatch, touches no state, and emits no logs. The sink address itself is the discriminator for the kind of metadata, so indexers extract attribution directly from the transaction by filtering `calls` on `to`, with no shared parser, tag byte, or off-chain registry.

## Motivation

Wallet builders and applications attach metadata to transactions for attribution and analytics. On legacy transaction types this metadata is appended to `tx.input` as trailing bytes:

- [ERC-8021](./eip-8021.md) builder codes identify which wallet builder constructed a transaction.
- A "data suffix" carries opaque app- or session-level metadata for attribution and analytics.

[EIP-8130](./eip-8130.md) splits execution into a structured `calls` array, so the single trailing-bytes location no longer exists in the same form. A replacement mechanism must satisfy several constraints:

- **Multiple independent authors.** A builder code, a data suffix, and future metadata kinds come from different parties. None should have to negotiate a shared encoding inside one field.
- **Cheap on rollups.** The marker must compress to near-nothing in rollup batches so attribution adds negligible L1 cost.
- **No execution side effects.** Metadata must not dispatch into a contract, mutate state, emit logs, or be confusable with a real call.
- **Self-contained indexing.** Everything an indexer needs should live in the transaction itself, with no sidecar fetch or registry lookup.

A reserved sink address per metadata kind satisfies all four. The constant sink address compresses away in rollup batches, so the marginal L1 cost approximates that of an opaque top-level field while gaining structured separation between kinds and an open namespace for new kinds.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Metadata sink addresses

A **metadata sink address** is a reserved 20-byte address that, when used as the `to` field of an [EIP-8130](./eip-8130.md) call, marks that call as a metadata record rather than an execution call.

This proposal reserves the following sink addresses:

| Metadata kind | Sink address |
| --- | --- |
| [ERC-8021](./eip-8021.md) builder codes | `0x8021802180218021802180218021802180218021` |
| Data suffix (opaque bytes) | `0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada7a` |

The sink address is the sole discriminator for the metadata kind. Indexers MUST NOT rely on a tag byte or in-band type prefix to distinguish kinds.

### Metadata call semantics

A call is a **metadata call** if and only if its `to` field equals a reserved metadata sink address. For every metadata call:

- The `value` field MUST be `0`.
- The `data` field carries the metadata payload for the kind associated with the sink address. For the [ERC-8021](./eip-8021.md) sink, `data` MUST be a valid [ERC-8021](./eip-8021.md) builder code payload. For the data suffix sink, `data` is opaque bytes whose interpretation is left to the producing application.
- The call MUST NOT be dispatched to EVM execution. Clients MUST treat a metadata call as a no-op with respect to state: no code is loaded or executed at the sink address, no storage is read or written, no logs are emitted, and no call frame is created.
- A metadata call MUST NOT revert and MUST NOT affect the success or failure of any other call in the `calls` array.

Metadata payload bytes are part of the transaction and are therefore covered by the sender signature, and, when present, the payer signature, under the [EIP-8130](./eip-8130.md) authentication model. Metadata payload bytes are charged at the [EIP-2028](./eip-2028.md) per-byte calldata rates, the same as any other bytes in the `calls` array.

### Wallet behavior

A wallet MAY include zero, one, or many metadata calls in a single transaction, mixing kinds freely. Wallets SHOULD append metadata calls to the end of the `calls` array, after all execution calls, to minimize interaction with applications that observe call ordering.

A wallet SHOULD include at most one metadata call per kind per transaction. If a wallet includes multiple metadata calls of the same kind, indexers process them as an ordered list (see [Indexer behavior](#indexer-behavior)).

### Indexer behavior

For every [EIP-8130](./eip-8130.md) transaction, an indexer:

1. Enumerates the `calls` array in order.
2. For each call, compares `call.to` against the set of known metadata sink addresses.
3. If `call.to` matches a sink address, records `call.data` as metadata of the kind associated with that address.
4. Otherwise treats the call as an execution call and processes it normally.

When multiple metadata calls share a kind, indexers process them as an ordered list and MAY deduplicate or aggregate per their own needs.

## Rationale

### Sink address as discriminator

Using the address itself as the kind discriminator means no shared framing or tag byte is needed across independent metadata authors. Adding a new metadata kind is a matter of reserving a new address; it requires no protocol change, no shared registry, and no parser coordination. This is the key advantage over a single top-level opaque metadata field, which forces every author to share one slot and agree on internal framing.

### Compression parity with a top-level field

The sink address is constant across every transaction that uses a given kind. Rollup batch compression deduplicates this repeated constant to near-nothing, so using a 20-byte address as the discriminator costs effectively the same L1 bytes as an opaque top-level field after compression, while gaining structured separation and an extensible namespace.

### Non-executing by construction

Routing metadata through reserved addresses that clients never dispatch guarantees metadata cannot have side effects: it cannot access state, emit logs, create a call frame, or revert into the execution path. This makes metadata safe to attach unconditionally and impossible to confuse with a real call by a misbehaving contract.

### Self-contained indexing

Because the marker and payload both live in the transaction's `calls` array, an indexer needs nothing beyond the transaction: no sidecar fetch, no registry lookup, no off-chain channel. Read the transaction, filter on `to`, done.

### Alternative considered: top-level metadata field

A single top-level `metadata` field (opaque bytes) on the [EIP-8130](./eip-8130.md) transaction was considered. It mirrors the legacy `tx.input` suffix and has the smallest footprint for a single small payload. It was rejected as the primary mechanism because it is single-tenant: a builder code and a data suffix would have to share one field and agree on framing, and there is no native way for multiple parties to contribute independent metadata or to add new kinds without renegotiating that shared encoding. The sink-address approach achieves comparable post-compression cost while supporting native multi-attribution.

### Vanity addresses

The reserved addresses use memorable byte patterns derived from the metadata kind (`0x8021…` for [ERC-8021](./eip-8021.md), `0xda7a…` for "data"). The patterns are mnemonic only; they carry no semantic meaning to clients, which treat any reserved sink address identically.

## Backwards Compatibility

This proposal applies only to [EIP-8130](./eip-8130.md) transactions and introduces no changes to legacy transaction types.

During the transition from the legacy `tx.input` suffix convention to [EIP-8130](./eip-8130.md), wallets and indexers will encounter both forms. Indexers SHOULD continue to parse trailing-bytes metadata on legacy transaction types while parsing metadata calls on [EIP-8130](./eip-8130.md) transactions. A wallet constructing an [EIP-8130](./eip-8130.md) transaction MUST emit attribution metadata as metadata calls rather than attempting to append trailing bytes, since the structured `calls` array has no trailing-bytes location.

## Security Considerations

### Reserved address collisions

A sink address MUST NOT be an account that anyone controls or expects to receive value or calls. Reserved sink addresses SHOULD be confirmed to have no deployed code and no expectation of use as a normal account on target chains before adoption. Because metadata calls carry `value` `0` and are never dispatched, a collision with an existing account does not transfer value to that account, but implementers MUST still confirm the canonical bytes to avoid ambiguity.

### Unverified metadata

Metadata payloads are attestations by the signer, not protocol-verified facts. A builder code or data suffix asserts only that the transaction's signer (and payer, if any) included those bytes. Consumers MUST NOT treat metadata as authenticated beyond the identity of the transaction signer, and MUST NOT grant trust or privileges based on metadata content without independent verification appropriate to the claim.

### No execution guarantees

Clients MUST guarantee that metadata calls are never dispatched. A client that incorrectly executes a metadata call could load code at a future-deployed sink address, creating an unexpected side effect. Conformant clients treat any call to a reserved sink address as a non-executing no-op regardless of whether code exists at that address.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
