# [L-02] Flash liquidation helper uses hardcoded 50% close factor, incorrectly rejecting valid full-liquidations

## Summary

The flash liquidation helper contract validates the liquidation amount against a hardcoded 50% close factor (`DEFAULT_LIQUIDATION_CLOSE_FACTOR = 5000`). The main router uses a dynamic close factor — 100% for small positions (debt or collateral < $2000 base currency) or severely undercollateralized positions, 50% otherwise. When a position qualifies for 100% liquidation, the helper incorrectly rejects the attempt with `LiquidationAmountTooHigh`, causing liquidators who rely on the helper for pre-validation to miss valid liquidation opportunities.

## Vulnerability Detail

`contracts/flash-liquidation-helper/src/validation.rs`, lines 60–68:
```rust
let max_liquidatable = total_debt_base
    .checked_mul(DEFAULT_LIQUIDATION_CLOSE_FACTOR)  // hardcoded 5000 = 50%
    .ok_or(KineticRouterError::MathOverflow)?
    .checked_div(BASIS_POINTS_MULTIPLIER)
    .ok_or(KineticRouterError::MathOverflow)?;
if debt_to_cover_base > max_liquidatable {
    return Err(KineticRouterError::LiquidationAmountTooHigh);
}
```

The main router's dynamic close factor logic (`contracts/kinetic-router/src/liquidation.rs`, `validate_close_factor`):
- Returns **100%** when `total_debt_base < MIN_CLOSE_FACTOR_THRESHOLD` ($2000) OR `total_collateral_base < MIN_CLOSE_FACTOR_THRESHOLD`
- Returns **100%** when HF is below the partial liquidation threshold
- Returns **50%** otherwise

### Scenario where this matters

1. A small borrower has $1500 in debt (below `MIN_CLOSE_FACTOR_THRESHOLD = $2000`)
2. Liquidator calls the flash liquidation helper with `debt_to_cover = 100%` of debt
3. Helper computes `max_liquidatable = $1500 * 50% = $750`
4. Helper rejects: `$1500 > $750` → `LiquidationAmountTooHigh`
5. Main router would have accepted: dynamic close factor = 100% for this position

The liquidator's transaction is blocked at the pre-validation stage even though the main router would process it correctly.

## Impact

Low — the flash liquidation helper is an off-chain pre-validation contract; the main router is unaffected. However, liquidators using the helper for simulation/validation will receive false rejections for valid full-liquidation scenarios on small positions, potentially causing them to miss liquidation opportunities or submit incorrectly sized transactions.

## Code Location

`contracts/flash-liquidation-helper/src/validation.rs`, lines 60–68

## Recommended Fix

Implement the same dynamic close factor logic in the helper, or accept the computed close factor as a parameter:

```rust
// Option 1: replicate dynamic close factor logic
let close_factor = if total_debt_base < MIN_CLOSE_FACTOR_THRESHOLD 
    || total_collateral_base < MIN_CLOSE_FACTOR_THRESHOLD {
    MAX_LIQUIDATION_CLOSE_FACTOR  // 10000 = 100%
} else {
    DEFAULT_LIQUIDATION_CLOSE_FACTOR  // 5000 = 50%
};

let max_liquidatable = total_debt_base
    .checked_mul(close_factor)
    .ok_or(KineticRouterError::MathOverflow)?
    .checked_div(BASIS_POINTS_MULTIPLIER)
    .ok_or(KineticRouterError::MathOverflow)?;
```
