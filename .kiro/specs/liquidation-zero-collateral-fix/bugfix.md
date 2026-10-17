# Bugfix Requirements Document

## Introduction

Liquidation routines in `src/vaults/liquidation.rs` divide reward balances by derived values that depend on the collateral ratio without first checking whether the collateral ratio is zero. When processing a fully liquidated position (where `collateral_value == 0` and `borrowed_value > 0`), the collateral ratio evaluates to zero, and any downstream division by that ratio causes the transaction to revert with an opaque low-level error rather than a descriptive custom error. The fix adds an explicit zero-collateral guard before proceeding to division and surfaces a new `ZeroCollateralRatio` error variant when the guard fires.

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN a liquidation is attempted on a position where `collateral_value` is zero and `borrowed_value` is greater than zero THEN the system computes a collateral ratio of zero and proceeds to divide without guarding, causing the transaction to revert with an opaque low-level error.

1.2 WHEN `liquidate()` is called with `collateral_value == 0` and `borrowed_value > 0` THEN the system returns a `DivisionByZero` error (discriminant 27) that gives callers no domain-level context about why the operation failed.

1.3 WHEN `auto_deleverage()` is called on a fully liquidated position (`collateral_value == 0`, `borrowed_value > 0`) THEN the system reaches division logic without a descriptive pre-condition check, producing the same opaque revert.

### Expected Behavior (Correct)

2.1 WHEN a liquidation is attempted on a position where `collateral_value` is zero and `borrowed_value` is greater than zero THEN the system SHALL return `ContractError::ZeroCollateralRatio` before any division is attempted.

2.2 WHEN `liquidate()` is called with `collateral_value == 0` and `borrowed_value > 0` THEN the system SHALL return `Err(ContractError::ZeroCollateralRatio)` immediately, providing callers with a clear, domain-specific error explaining that the collateral ratio is zero.

2.3 WHEN `auto_deleverage()` is called on a fully liquidated position (`collateral_value == 0`, `borrowed_value > 0`) THEN the system SHALL return `Err(ContractError::ZeroCollateralRatio)` before any arithmetic on the position proceeds.

### Unchanged Behavior (Regression Prevention)

3.1 WHEN `liquidate()` is called with `collateral_value > 0` and `borrowed_value > 0` THEN the system SHALL CONTINUE TO compute the health factor and apply liquidation logic as before.

3.2 WHEN `liquidate()` is called with `borrowed_value == 0` THEN the system SHALL CONTINUE TO return `Ok` with `health_factor == u128::MAX` and `liquidated == false`.

3.3 WHEN `health_factor()` is called with a healthy position (`collateral_value >= borrowed_value`, both non-zero) THEN the system SHALL CONTINUE TO return the correct basis-point ratio without error.

3.4 WHEN `auto_deleverage()` is called on a position with `health_factor >= DISTRESSED_THRESHOLD_BPS` or `borrowed_value == 0` THEN the system SHALL CONTINUE TO return a non-deleveraged result unchanged.

3.5 WHEN `liquidate_at_twap()` is called with valid oracle prices and a position with positive collateral THEN the system SHALL CONTINUE TO price the position correctly and delegate to `liquidate()` without interference from the new guard.

3.6 WHEN `conserves_collateral()` is called on any `LiquidationResult` THEN the system SHALL CONTINUE TO enforce the zero-loss invariant: liquidator reward plus protocol reserve equals the purchase collateral for a liquidated result, and both are zero for a non-liquidated result.
