---
title: Parallel Execution Maps
description: Optionally include list of transactions without dependencies.
author: Justin Florentine (@jflo) <justin@florentine.us>
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2025-05-22
requires: 
---

## Abstract

With the advent of parallel transaction execution in some execution clients, it is now possible to share the safe execution path with other clients. When a client that supports parallel execution composes a block, and they know the set of independent transactions which can be executed out of order, they can share that set with all other clients which can then benefit from a parallel execution path.

## Motivation

On average, up to 70% of the transactions in a block are often independent of all the others. This large subset of the block may be executed in any order and still arrive at the same state root. The difficult part for clients is determining which transactions these are. 

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Constants

| Name                  | Value | Comment                                                          |
|-----------------------|-------|------------------------------------------------------------------|
| `METADATA_MAX_LENGTH` | 2048  | Maximum number of bytes allowed in the body of a metadata object |

This EIP introduces a new block header field designed akin to the EIP-7685 request type. The block header is appended to include an RLP list of typed `metadata` tuples:

A `metadata` object consists of a `metadata_type` byte prepended to an opaque byte array
`metadata_body`. The `metadata_body` field is limited to METADATA_MAX_LENGTH bytes.

| Name                      | Value  | Comment                               |
|---------------------------|--------|---------------------------------------|
| `EXEC_HINT_METADATA_TYPE` | `0x01` | The type prefix for execution hints   |

The encoding of an execution hint request is as follows. 

```python
metadata_type = EXEC_HINT_METADATA_TYPE
request_data = rlp(txIndex, ...)
```

Where `txIndex` is an integer, 0 based index into the block transaction list.

Clients MAY interpret the list of `txIndex` as the set of transactions in the block which may be executed in any order.

Clients MUST execute the remainder of the transactions in the block, which do not appear in the request, in the order they appear in the block.

Clients MUST NOT include this metadata if none of the transactions in the block are independent.

In the case of an incorrect stateroot after parallel execution, the block is treated exactly as invalid as if it had been executed serially.

## Rationale

Modern path-based data stores in ethereum clients are better suited to tracking the read/write state of storage slots, however they are not implemented consistently across all clients, and even if they were determining transaction independence is work that could easily be re-used across the network.

Entities such as MEV builders and searchers stand to extract more value if they know their blocks will be executed faster.



## Backwards Compatibility

The addition of another field to the blockheader requires a hardfork.

## Test Cases

TBD

## Reference Implementation

TBD

## Security Considerations

Proposers could include false execution maps, resulting in wasted work among clients who choose to use them.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
