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

[EIP-8130](./eip-8130.md) replaces the single `tx.input` byte string of legacy transactions with a structured `calls` array of execution phases, leaving transaction *metadata* — most commonly a **data suffix**, including [ERC-8021](./eip-8021.md) builder codes — without a home. This proposal reserves a single, codeless **metadata sink address**. A call in the transaction's `calls` whose `to` field equals the sink address is a metadata record: its `data` carries signed metadata and, because the address has no code (and [EIP-8130](./eip-8130.md) calls carry no value), the call is a guaranteed no-op that a node MAY skip dispatching. The metadata stays in the signed transaction calldata regardless of execution outcome, and indexers read it by filtering `calls` on `to`. A data suffix is the conventional form of this metadata, and [ERC-8021](./eip-8021.md) builder codes are a structured data suffix.

## Motivation

Wallets and applications attach metadata to transactions for attribution and analytics. On legacy transaction types this metadata is a **data suffix** — extra bytes appended to the end of `tx.input`, a convention that predates [ERC-8021](./eip-8021.md). [ERC-8021](./eip-8021.md) defines a *structure* for such bytes so that parsers can read builder codes consistently. The terms nest: an [ERC-8021](./eip-8021.md) builder code is a data suffix, and a data suffix is metadata.

[EIP-8130](./eip-8130.md) splits execution into a structured `calls` array — a list of phases, each an ordered, atomic batch of `[to, data]` calls — so the trailing-bytes location no longer exists. This structure is a natural carrier: an [EIP-8130](./eip-8130.md) call is just a `to` and `data` and carries no value at all, so a metadata record is simply a call to a reserved address — no new top-level transaction field, and nothing that can be confused with a value transfer. Because the reserved address is constant, it compresses away in rollup batches, so the marginal L1 cost approximates that of an opaque trailing field while keeping the metadata inside the structured, individually signed transaction body.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Metadata sink address

This proposal reserves a single, codeless 20-byte address as the **metadata sink**:

| Purpose | Sink address |
| --- | --- |
| Transaction metadata (a data suffix, including [ERC-8021](./eip-8021.md) builder codes) | `0xda7ada7ada7ada7ada7ada7ada7ada7ada7ada7a` |

A **metadata call** is a call within an [EIP-8130](./eip-8130.md) transaction's `calls` (in any phase) whose `to` field equals the sink address. Its `data` field is the metadata payload: arbitrary opaque bytes. Producers that want structured, consistently parseable attribution SHOULD format the payload as an [ERC-8021](./eip-8021.md) data suffix, which is self-describing and needs no additional tag; otherwise the bytes are interpreted per an application convention.

### Scope

Only calls that appear directly in the transaction's `calls` (the signed phases) are metadata. A call to the sink address that originates from within EVM execution (an internal call or subcall made while a call is dispatched) is not a metadata record and MUST NOT be indexed as one. This proposal does not change how metadata is parsed on any non-[EIP-8130](./eip-8130.md) transaction type.

### Metadata call semantics

For every metadata call:

- The `data` field is the metadata payload, as described in [Metadata sink address](#metadata-sink-address). [EIP-8130](./eip-8130.md) calls carry no value, so a metadata call is inherently value-less.
- A metadata call has no execution side effects. The sink address MUST remain codeless (see [Security Considerations](#security-considerations)), so dispatching the call under standard [EIP-8130](./eip-8130.md) semantics is an immediate no-op: no code runs, no storage is touched, no logs are emitted, and the call cannot revert into or alter any other call.

Metadata payload bytes are part of the transaction, covered by the sender signature (and the payer signature, when present) under the [EIP-8130](./eip-8130.md) authentication model, and charged at the [EIP-2028](./eip-2028.md) per-byte calldata rates like any other bytes in `calls`.

### Dispatch and gas

Because the sink address is codeless, a metadata call is a guaranteed no-op, so a node MAY choose not to dispatch it — recording the `to` and `data` without creating a call frame — to save the work and gas of dispatching to an empty account. Dispatched or not, the metadata bytes remain in the signed transaction and the observable state is identical.

A metadata call SHOULD NOT affect execution and, because a node MAY skip dispatch, SHOULD NOT push an otherwise-valid transaction into out-of-gas. The metadata remains indexable from the signed `calls` regardless of whether the transaction's execution phases succeed, revert, or are skipped. Turning the skipped dispatch into a reduced charge to the sender (rather than a local node optimization) requires the protocol-level treatment in [Convention versus protocol-level support](#convention-versus-protocol-level-support).

### Wallet behavior

A data suffix that describes the transaction as a whole — most importantly an [ERC-8021](./eip-8021.md) builder code — SHOULD be placed in its own dedicated phase containing only that single metadata call, appended as the last phase of `calls`. Isolating it in its own phase keeps it out of the atomic execution phases and the work they do: it cannot cause an execution phase to revert, and if an earlier phase reverts (skipping later phases) the metadata is simply never dispatched while remaining present in the signed `calls` for indexing. A transaction MUST contain at most one metadata call carrying an [ERC-8021](./eip-8021.md) builder code; a transaction has a single builder.

Positional metadata — a payload that annotates a specific call, such as a per-transfer memo — MAY instead be placed as a metadata call within an execution phase, adjacent to the call it annotates. Call ordering MAY be significant to consumers, so wallets SHOULD order such metadata calls to preserve the intended association.

### Indexer behavior

For every [EIP-8130](./eip-8130.md) transaction, an indexer enumerates `calls` phase by phase, in order, and for each call:

1. If `call.to` equals the sink address, records `call.data` as metadata. If the payload is [ERC-8021](./eip-8021.md)-formatted, the indexer extracts the builder code per [ERC-8021](./eip-8021.md); otherwise it interprets the bytes per its own convention.
2. Otherwise treats the call as an execution call and processes it normally.

Indexers process multiple metadata calls as an ordered list and SHOULD preserve each one's position relative to the surrounding execution calls, so a positional payload (such as a per-transfer memo) can be associated with the adjacent call. Metadata MUST be read from the signed `calls` regardless of per-phase execution status, since a metadata call may be skipped (for example, in a trailing phase after an earlier revert) yet still validly committed by the signer.

## Rationale

### A single sink address

A single sink keeps the metadata location consistent with how indexers already pull [ERC-8021](./eip-8021.md) builder codes from trailing calldata: the same [ERC-8021](./eip-8021.md) parser applies to the `data` of a metadata call. Because [ERC-8021](./eip-8021.md) payloads are self-describing, no per-kind address or in-band tag is needed to tell a builder code from other metadata. Per-kind sink addresses were considered and rejected: they add addresses to coordinate while duplicating structure [ERC-8021](./eip-8021.md) already provides.

### Reusing the `calls` array instead of a top-level field

A top-level `dataSuffix` field was considered. It keeps the familiar name but widens the transaction type, and after rollup compression costs effectively the same as a constant sink address. The [EIP-8130](./eip-8130.md) `calls` array is already the right shape: entries are ordered, individually addressed, carry their own `data`, and need no value — so a metadata record reuses the existing structure rather than adding a field. The data suffix concept is preserved as the conventional payload carried at the sink.

### Convention versus protocol-level support

As specified, this is an application-layer convention: a metadata call is an ordinary [EIP-8130](./eip-8130.md) call to a reserved address that dispatches to an empty account as a no-op, so the sender still pays the small dispatch cost on top of the [EIP-2028](./eip-2028.md) calldata cost. A follow-on **EIP** could make clients recognize the sink address and skip dispatch entirely — charging only for calldata — and optionally emit a log per metadata call for cheaper, event-based indexing. The on-the-wire shape of a metadata call is identical under both, so adoption can begin as a convention and tighten into protocol behavior later.

### Vanity address

The reserved address uses a mnemonic byte pattern (`0xda7a…` for "data"). It carries no meaning to clients, which treat the sink like any other codeless account.

## Backwards Compatibility

This proposal applies only to [EIP-8130](./eip-8130.md) transactions and changes nothing on legacy transaction types. During the transition, indexers SHOULD continue to parse trailing-bytes data suffixes (including [ERC-8021](./eip-8021.md) builder codes) on legacy transactions while parsing metadata calls on [EIP-8130](./eip-8130.md) transactions. The payload format is unchanged — the same [ERC-8021](./eip-8021.md) bytes, relocated from `tx.input` to a metadata call's `data` — so existing [ERC-8021](./eip-8021.md) parsers apply directly. A wallet constructing an [EIP-8130](./eip-8130.md) transaction MUST emit metadata as a metadata call, since the `calls` array has no trailing-bytes location.

## Security Considerations

### The sink address must remain codeless

As an application-layer convention, a metadata call is a no-op *only because the sink address has no code*. If code were ever deployed there, a metadata call would execute it, turning a metadata record into a real call with side effects. Therefore the sink address MUST be confirmed codeless on every target chain before adoption and MUST be chosen so no party is expected to control or deploy to it (the vanity pattern is not a `CREATE`/`CREATE2` output and has no known private key); adopters SHOULD monitor it for unexpected code. The protocol-level enhancement removes this risk by skipping dispatch regardless of code presence. Because [EIP-8130](./eip-8130.md) calls carry no value, a collision transfers nothing to the sink, but implementers MUST still confirm the canonical address bytes.

### Unverified metadata

Metadata is an attestation by the signer, not a protocol-verified fact: it asserts only that the signer (and payer, if any) committed to those bytes. Consumers MUST NOT treat metadata as authenticated beyond the transaction signer's identity, MUST NOT grant trust or privileges based on its content without independent verification, and MUST sanitize untrusted payload bytes before use.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
