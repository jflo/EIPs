---
title: Parallel Execution Maps
description: Optionally include list of transactions without dependencies.
author: Justin Florentine (@jflo) <justin@florentine.us>
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2025-05-22
requires: [EIP-7685](eip-7685.md)
---

## Abstract

With the advent of parallel transaction execution in some execution clients, it is now possible to share the safe execution path with other clients. When a client that supports parallel execution composes a block, and they know the set of independent transactions which can be executed out of order, they can share that set with all other clients which can then benefit from a parallel execution path.

## Motivation

On average, up to 70% of the transactions in a block are often independent of all the others. This large subset of the block may be executed in any order and still arrive at the same state root. The difficult part for clients is determining which transactions these are. 

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

This EIP introduces a new EIP-7685 request type `0x03` that clients MUST NOT treat as an error upon encountering.

| Name                     | Value  | Comment                                                       |
|--------------------------|--------|---------------------------------------------------------------|
| `EXEC_HINT_REQUEST_TYPE` | `0x03` | The [EIP-7685](./eip-7685.md) type prefix for execution hints |

The [EIP-7685](./eip-7685.md) encoding of an execution hint request is as follows. 

```python
request_type = EXEC_HINT_REQUEST_TYPE
request_data = rlp(txIndex, ...)
```

Where `txIndex` is an integer, 0 based index into the block transaction list.

Clients MAY interpret the list of `txIndex` as the set of transactions in the block which may be executed in any order.

Clients MUST execute the remainder of the transactions in the block, which do not appear in the request, in the order they appear in the block.

Clients MUST NOT include this request if none of the transactions in the block are independent.

In the case of an incorrect stateroot after parallel execution, clients MUST re-execute all transactions in the block in block order, to confirm the bad block before reporting it as such.

## Rationale



## Backwards Compatibility

The use of general purpose requests in this design ensures maximal backward compatibility. The optional nature of the execution hint means clients may choose to ignore the entire request.

## Test Cases

TBD

## Reference Implementation

TBD

## Security Considerations

Proposers could include false execution maps, resulting in wasted work among clients who choose to use them.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
