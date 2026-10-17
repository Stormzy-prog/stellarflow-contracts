# Bugfix Requirements Document

## Introduction

`AirdropDistributor.sol` contains a custom Merkle proof verifier that hashes internal node pairs
using `keccak256(abi.encodePacked(nodeA, nodeB))` without first sorting the two child hashes.
Because the order of `nodeA` and `nodeB` is determined entirely by the caller-supplied proof path,
an attacker can craft a proof that presents the same two leaf/node values in the opposite order and
produce an identical internal hash. This is a classic **second-preimage** (also called
"hash-order") vulnerability: a single valid leaf can be promoted to appear as an internal node, or
two different proofs can verify against the same root, undermining the integrity guarantee of the
Merkle tree and allowing double-claiming or unauthorised airdrop claims.

The fix has two parts:
1. Sort every sibling pair (`if (a > b) swap(a, b)`) before hashing so the hash is always
   deterministic regardless of proof ordering.
2. Delegate proof verification to OpenZeppelin's audited `MerkleProof` library, which already
   implements sorted-pair hashing, eliminating the need for a hand-rolled verifier.

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN a Merkle proof supplies sibling hashes in reverse order (i.e. `nodeB < nodeA`) THEN the
    system hashes the pair as `keccak256(abi.encodePacked(nodeA, nodeB))` without sorting, producing
    a hash that differs from the canonical tree hash and therefore either rejects a valid claim or
    accepts a forged one depending on how the tree was originally built.

1.2 WHEN a caller supplies a crafted proof in which a previously verified leaf value is reused as an
    internal node THEN the system accepts the proof as valid, enabling a second-preimage attack and
    allowing an address to claim tokens it is not entitled to.

1.3 WHEN two proofs for different recipients are submitted where one proof's intermediate hash equals
    another's leaf hash (unaligned tree structure) THEN the system cannot distinguish the two cases
    and may grant duplicate or unauthorised claims.

### Expected Behavior (Correct)

2.1 WHEN a Merkle proof supplies sibling hashes in any order THEN the system SHALL sort the pair
    (`min, max`) before hashing, producing `keccak256(abi.encodePacked(min, max))` and always
    arriving at the canonical root regardless of proof-path direction.

2.2 WHEN a caller supplies a crafted proof that attempts to reuse a leaf value as an internal node
    THEN the system SHALL reject the proof as invalid, preventing second-preimage exploitation.

2.3 WHEN any Merkle proof is verified THEN the system SHALL delegate verification to
    OpenZeppelin's `MerkleProof.verify` (or `verifyCalldata`) function, which implements
    sorted-pair hashing and is independently audited.

### Unchanged Behavior (Regression Prevention)

3.1 WHEN a legitimate claimant submits a valid proof for an address and amount that are genuinely
    present in the Merkle tree THEN the system SHALL CONTINUE TO verify the proof successfully and
    allow the claim to proceed.

3.2 WHEN an address has already claimed its airdrop allocation THEN the system SHALL CONTINUE TO
    reject subsequent claim attempts from that same address.

3.3 WHEN a proof is supplied for an address-and-amount pair that is not in the Merkle tree THEN the
    system SHALL CONTINUE TO reject the claim as invalid.

3.4 WHEN a valid claim is processed THEN the system SHALL CONTINUE TO transfer the correct token
    amount to the claimant and emit the appropriate event.

---

## Bug Condition Pseudocode

### Bug Condition Function

```pascal
FUNCTION isBugCondition(X)
  INPUT:  X of type ProofInput { proof: bytes32[], account: address, amount: uint256 }
  OUTPUT: boolean

  // True when the proof contains at least one sibling pair supplied in descending order,
  // OR when a leaf hash value appears as an intermediate node in the proof path.
  siblingOutOfOrder ← EXISTS i IN X.proof SUCH THAT
                        computedHash(X, i-1) > X.proof[i]   // caller controls ordering
  leafReusedAsNode  ← EXISTS i IN X.proof SUCH THAT
                        X.proof[i] = keccak256(keccak256(abi.encode(account, amount)))

  RETURN siblingOutOfOrder OR leafReusedAsNode
END FUNCTION
```

### Property: Fix Checking

```pascal
// For all inputs that match the bug condition, the fixed function must reject or
// canonically resolve the proof rather than producing an incorrect acceptance/rejection.
FOR ALL X WHERE isBugCondition(X) DO
  result ← AirdropDistributor'(X)   // F' = fixed implementation
  ASSERT (result = VALID) IFF (X represents a genuine tree membership)
  ASSERT no second-preimage accepted
END FOR
```

### Property: Preservation Checking

```pascal
// For all inputs that do NOT match the bug condition (well-formed proofs in correct order,
// no leaf reuse), the fixed implementation must behave identically to the original.
FOR ALL X WHERE NOT isBugCondition(X) DO
  ASSERT AirdropDistributor(X) = AirdropDistributor'(X)
END FOR
```
