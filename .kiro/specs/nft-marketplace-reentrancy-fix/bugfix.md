# Bugfix Requirements Document

## Introduction

In `src/NFTMarketplaceRouter.sol`, mint-and-deposit listing routines invoke `_safeMint()` before the corresponding listing storage record has been written. Because `_safeMint()` triggers the `onERC721Received` callback on the token recipient, an attacker-controlled receiver contract gains execution control before the listing record exists. At that moment the marketplace has no authoritative on-chain record binding the seller to the listing, so the callback can re-enter the router's listing entry points, bypass seller-authorization checks, and manipulate listing state.

The fix applies the **Check-Effects-Interactions** (CEI) pattern — writing the listing storage record before any external call — and adds a `nonReentrant` modifier to listing minting entry points so that any attempted re-entry during the callback is hard-rejected at the guard level.

---

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN a mint-and-deposit listing routine is called THEN the system invokes `_safeMint()` before writing the listing storage record, triggering `onERC721Received` on the recipient while the record is absent.

1.2 WHEN `onERC721Received` fires on an attacker-controlled receiver during a mint-and-deposit call THEN the system allows re-entry into listing minting entry points because no `nonReentrant` guard is present.

1.3 WHEN a listing entry point is re-entered via the `onERC721Received` callback before the listing record exists THEN the system fails to validate seller authorization, enabling the attacker to register fraudulent listings or manipulate existing listing state.

### Expected Behavior (Correct)

2.1 WHEN a mint-and-deposit listing routine is called THEN the system SHALL write the listing storage record to persistent state before invoking `_safeMint()` or any other external call.

2.2 WHEN `onERC721Received` fires on any recipient during a mint-and-deposit call THEN the system SHALL reject any re-entrant call into a guarded listing entry point with a `ReentrancyGuardReentrantCall` error, leaving all storage state unchanged.

2.3 WHEN a listing entry point protected by `nonReentrant` is called THEN the system SHALL atomically set the reentrancy lock before executing any logic and release it only after all state transitions and external calls complete.

### Unchanged Behavior (Regression Prevention)

3.1 WHEN a mint-and-deposit listing routine is called by an authorized seller with valid inputs and no concurrent re-entrant call is in progress THEN the system SHALL CONTINUE TO create the listing record and mint the NFT successfully, emitting the expected listing event.

3.2 WHEN a buyer queries an active listing record that was created through the fixed mint-and-deposit routine THEN the system SHALL CONTINUE TO return accurate listing metadata, including correct seller address, token ID, and price.

3.3 WHEN a non-listing entry point (e.g., purchase, cancel, withdraw) is called on the marketplace THEN the system SHALL CONTINUE TO operate with existing behavior unaffected by the CEI reordering applied solely to listing minting entry points.

3.4 WHEN `_safeMint()` is called within the fixed routine and the recipient's `onERC721Received` hook returns the expected selector THEN the system SHALL CONTINUE TO complete the mint and finalize the listing without reverting.

---

## Bug Condition Derivation

**Bug Condition Function:**

```pascal
FUNCTION isBugCondition(X)
  INPUT: X of type ListingMintCall
  OUTPUT: boolean

  // True when a mint-and-deposit entry point is executing AND
  // the listing storage record has not yet been written for X.tokenId
  RETURN X.entryPoint ∈ LISTING_MINT_ENTRY_POINTS
     AND listingRecord(X.tokenId) = EMPTY
     AND _safeMint_not_yet_called(X) = FALSE   -- i.e. _safeMint has been called
END FUNCTION
```

**Property: Fix Checking**

```pascal
// For every mint-and-deposit call that previously triggered the bug,
// the listing record MUST exist before any external call is dispatched.
FOR ALL X WHERE isBugCondition_pre_fix(X) DO
  result ← listMintAndDeposit'(X)
  ASSERT listingRecord(X.tokenId) ≠ EMPTY BEFORE _safeMint_called(X)
  ASSERT no_reentrant_success(result)
END FOR
```

**Property: Preservation Checking**

```pascal
// For all calls that do NOT match the bug condition (no reentrancy, valid inputs),
// the fixed implementation must behave identically to the original.
FOR ALL X WHERE NOT isBugCondition(X) DO
  ASSERT listMintAndDeposit(X) = listMintAndDeposit'(X)
END FOR
```
