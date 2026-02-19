---  
eip: TBD  
title: LUCID - Encrypted Mempools with Unconditional Inclusion Lists  
description: A mechanism for pre-confirming encrypted transactions via unconditional inclusion lists, executed at the top of the subsequent block.  
author: discussions-to: https://ethereum-magicians.org/t/frame-transaction/27617  
status: Draft  
type: Standards Track  
category: Core  
created: 2026-02-04  
requires: 7732, 7805, 2718
---  

## Abstract
This proposal introduces LUCID, a protocol upgrade that implements a trustless encrypted mempool. It extends EIP-7805 (FOCIL) by allowing includers to propose Sealed Transactions (STs) that the builder must commit to.


Includers determine the priority order of their lists, and builders are mandated to include these transactions up to a hard gas limit, providing strong censorship resistance and simplified gas accounting.

## Motivation
Transactions remain encrypted during the inclusion process, preventing front-running and sandwiching by builders.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Constants and Parameters

```python
TOB_GAS_FRACTION_DENOMINATOR = 8
ToB_gas_limit = block.gas_limit // TOB_GAS_FRACTION_DENOMINATOR
MAX_ST_COMMITS_PER_IL = 4
MAX_ST_COMMITS = MAX_ST_COMMITS_PER_IL*IL_COMMITTEE_SIZE
```

### Sealed transactions

A sealed transaction (ST) is an envelope sent from wallets which contains:

1. a signed ST ticket transaction (to later be included on-chain), and
2. an `encrypted_tx` ciphertext envelope (not included on-chain).

It is encoded in SSZ when being included in FOCIL style Inclusion Lists.

```python
class SealedTransaction(Container):
    ticket: Transaction                  # EIP-2718, ST_TICKET_TX_TYPE
    ciphertext_envelope: ByteList[MAX_BYTES_PER_TRANSACTION]
```

It is encoded in RLP when propagating across the mempool, as a new EIP-2718 transaction.

```
ST_TX_TYPE || rlp([
  st_ticket,
  ciphertext_envelope
])
```

#### ST ticket

An ST ticket is a new EIP-2718 typed transaction. It is charged and consumes the sender’s execution nonce, but is not executed as a normal EVM message call.

```
ST_TICKET_TX_TYPE || rlp([
    chain_id,
    nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    decryptor_address,
    decryptor_fee,
    reveal_commitment,
    ciphertext_hash,
    signature_y_parity,
    signature_r,
    signature_s
])
```

Besides regular transaction fields, the ST ticket also has a:

* `decryptor_address` of the entity responsible for decryption,
* `decryptor_fee` that is a fee for timely decrypting the plaintext transaction,
* `reveal_commitment` (see below),
* `ciphertext_hash`, where `ciphertext_hash = keccak256(ciphertext_envelope)` binds the ciphertext bytes.

#### Ciphertext envelope

The `ciphertext_envelope` of the ST is defined as:

```
rlp([header_len:u16 || header || dem_iv || dem_ciphertext])
```

The `header_len` is big-endian. The `header` is `header_len` bytes and is opaque to the protocol. This field is reserved for future use.

The `dem_ciphertext` is the output of a DEM `ChaCha20-Poly1305` (RFC 8439) AEAD encryption with:

* DEM key – `k_dem` (32 bytes).
* IV – `dem_iv` (12 bytes).
* Empty `aad` (`b""`).
* `dem_ciphertext` is `ciphertext || tag` (`tag` is 16 bytes).

The `ciphertext_envelope` decrypts to `prioritized_tx` which contains   a top-of-block (ToB) fee `ToB_fee_per_gas`, and a `plaintext_tx` EIP-1559 transaction to be executed onchain.

```
rlp([
    ToB_fee_per_gas: uint64,
    plaintext_tx: Transaction  # Type 2 transaction

])
```

The `plaintext_tx` must have:

* `max_priority_fee_per_gas = 0`
* `max_fee_per_gas = 0`
* `gas_limit = ticket.gas_limit`, where the `ticket` transaction has the same ciphertext

The `gas_limit` must be sufficient to cover the calldata cost of the byte size of the `SealedTransaction` and is thus bounded by the EIP-7825 transaction gas limit.

The `reveal_commitment` binds the revealed plaintext payload to a specific ticket and to the ToB fee. `ticket_from` is the recovered sender address from the ticket signature.

```python
class RevealCommitmentPreimage(Container):
    chain_id: uint256
    ticket_from: ExecutionAddress
    ticket_nonce: uint64
    plaintext_tx: Transaction
    ToB_fee_per_gas: uint64

reveal_commitment = hash_tree_root(RevealCommitmentPreimage(...))
```


### Inclusion lists with STs

The `InclusionList` container of EIP-7805 is extended with a list of STs to provide CR:

```python
class InclusionList(Container):
    ...
    sealed_transactions: List[SealedTransaction, MAX_ST_COMMITS_PER_IL]
```

Attesters enforce that the STs of each timely IL are unconditionally included in the payload. The STs of each IL have an aggregate gas limit of `ST_gas_limit_per_IL = ToB_gas_limit // IL_COMMITTEE_SIZE` of ToB gas.

### Engine API Changes

- Add `engine_getInclusionListV2` endpoint to retrieve an IL which may contain STs from the ExecutionEngine.
- Add `engine_updatePayloadWithInclusionListV2` endpoint to update a payload with the IL that may contain STs that should be used to build the block. This takes as an argument an 8-byte `payloadId` of the ongoing payload build process, along with the IL itself.
- Add `engine_newPayloadV6` endpoint which consumes an `ExecutionPayloadv5` to accept decryption keys.

### Beacon Chain RPC API Changes
- Update `/eth/v2/events` endpoint to notify clients that their ST has been included, and it is safe to publish the key.
- Add `/eth/v1/beacon/key` endpoint to relay the key to the libp2p topic. See section on Key message below.

### Extended payload bid

The `ExecutionPayloadBid` is extended with a list of "ST-commitments", canonical IL roots and `key_adherence` bits. The `ticket_hash` is the `keccak256` of the ST ticket transaction bytes. Each ST-commitment specifies four fields:

```python
class STCommitment(Container):
    commitment_root: Bytes32 # The ticket_hash
    gas_obligation: uint64   # The gas_limit of the ST ticket

class ExecutionPayloadBid(Container):
    ...
    commits: List[STCommitment, MAX_ST_COMMITS]
    key_adherence: List[Boolean, MAX_ST_COMMITS]
    IL_roots: List[Bytes32, IL_COMMITTEE_SIZE]
```

### Execution payload expansion

The `DecryptedTransaction` adds a `ticket_index` to the `RevealedTransaction`:

```python
class DecryptedTransaction(Container):
    ticket_index: uint32       # index into the parent block's st_tickets list
    plaintext_tx: Transaction  # decoded type-2 tx with fixed fee fields
    ToB_fee_per_gas: uint64
```

The execution payload is prepended with the `st_tickets` and the `decrypted_transactions`:

```python
class ExecutionPayload(Container):
    st_tickets: List[Transaction, MAX_ST_COMMITS]
    decrypted_transactions: List[DecryptedTransaction, MAX_ST_COMMITS]
    ...
```

### Key message

Decryptors publish a per-ticket DEM key `k_dem` in a signed message.

```python
class LucidKeyMessage(Container):
    chain_id: uint256
    scheduling_beacon_block_root: Bytes32
    scheduling_slot: uint64
    commit_index: uint8          # index of the ST-commitment
    k_dem: Bytes32
    signature: Bytes65
```

The `LucidKeyMessage` is valid if:

1. It is signed by the `decryptor_address` of the corresponding ST ticket.
2. Its `chain_id` matches the chain.
3. It references the correct scheduling block and a scheduled commitment with  `commit_index`.
4. The client has obtained the corresponding `SealedTransaction` off-chain (from IL data/gossip), and decrypting `encrypted_tx` using `k_dem` with `ChaCha20-Poly1305` and associated data yields a `RevealedTransaction` whose:
   * `hash_tree_root(RevealCommitmentPreimage(...)) == ticket.reveal_commitment`, and
   * `plaintext_tx` decodes to a type-2 (EIP-1559) transaction with:
      * `max_fee_per_gas = 0`,
      * `max_priority_fee_per_gas = 0`,
      * `gas_limit = ticket.gas_limit`.

The plaintext transaction sender and nonce do not need to match the ticket sender and nonce.

### Key timeliness

Each PTC member publishes a signed vote message indicating which valid keys, scheduled for the current slot, that they have observed:

```python
class LucidKeyTimelinessVote(Container):
    chain_id: uint256
    scheduling_beacon_block_root: Bytes32
    scheduling_slot: uint64
    keys_observed: Bitlist[MAX_ST_COMMITS]
    signature: BLSSignature
```

A bit `keys_observed[j]` is set to 1 iff the voter observed, by the key observation deadline of the PTC, a valid `LucidKeyMessage` for `(scheduling_beacon_block_root, scheduling_slot, commit_index=j)`. Nodes propagate equivocated `LucidKeyTimelinessVote` messages the same way they do for equivocated ILs and keys. Attesters store received votes at the `lucid_key_vote_deadline`, but keep listening for new messages and equivocation until they attest.

Let `timely_key_votes` be a bitfield of observed votes on a key at `lucid_key_vote_deadline` and `total_key_votes` be a bitfield of observed votes on a key at the attestation time. The number of committee members that equivocated are tallied in `equivocated_votes`. These votes are not included in the key votes bitfields. Further define:

* `timely_1 = sum(timely_key_votes)`,
* `timely_0 = len(timely_key_votes) - timely_1`,
* `late_1 = sum(total_key_votes) - timely_1`,
* `late_0 = len(total_key_votes) - sum(total_key_votes) - timely_0`

The attester considers:

* All non-equivocated votes it observed at `lucid_key_vote_deadline` to have been seen by the builder.
* All non-equivocated votes it observed after `lucid_key_vote_deadline` to have potentially been seen by the builder.
* All equivocated votes to potentially have been seen voting either `0` or `1`.

The builder can thus treat a key as missing (`key_adherence[i] = 0`) under condition:

* `2 * timely_1 <= (timely_1 + timely_0 + late_0 + equivocated_votes)` ;

and treat a key as observed (`key_adherence[i] = 1`) under condition:

* `2 * (timely_1 + late_1 + equivocated_votes) > (timely_1 + timely_0 + late_1 + equivocated_votes)`.

If this test fails for any key, the attester votes against the block.


## Security Concerns
HPKE is being newly considered in this EIP, will need a broad review.

## Economic Concerns
Any new refund mechanics should be closely examined for game-ability.


