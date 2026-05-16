# [L-01] `swap_collateral` uses `percent_mul` (round-down) instead of `percent_mul_up` for protocol fee, causing consistent fee underpayment

## Summary

In `swap_collateral`, the protocol fee is calculated with `percent_mul` (which truncates/rounds down), while every other protocol fee path in the codebase uses `percent_mul_up` (which rounds up, favoring the protocol). This causes the protocol to collect one unit less fee per swap whenever the fee calculation does not divide evenly, resulting in a small but systematic underpayment.

## Vulnerability Detail

`contracts/kinetic-router/src/swap.rs`, line 197:
```rust
let protocol_fee = k2_shared::utils::percent_mul(to_amount_received, swap_config.flash_loan_premium_bps)?;
```

All other protocol fee calculations use `percent_mul_up`:

| Location | Function | Rounding |
|---|---|---|
| `flash_loan.rs:86` | Flash loan premium | `percent_mul_up` |
| `liquidation.rs:358` | Liquidation protocol fee | `percent_mul_up` |
| `swap.rs:197` | Swap collateral fee | `percent_mul` ← inconsistent |

`percent_mul` computes `(value * percentage + HALF_BASIS_POINTS) / BASIS_POINTS_MULTIPLIER` — standard half-up rounding. `percent_mul_up` computes `(value * percentage + BASIS_POINTS_MULTIPLIER - 1) / BASIS_POINTS_MULTIPLIER` — ceiling rounding.

The difference: when `value * percentage` is not divisible by `BASIS_POINTS_MULTIPLIER`, `percent_mul` rounds to nearest while `percent_mul_up` always rounds up. For fee collection, rounding up is the protocol-favorable direction.

## Impact

Low — the protocol collects at most 1 unit less fee per swap. Over many swaps this is a small but consistent loss. No user funds are at risk; the underpayment goes to the user performing the swap.

## Code Location

`contracts/kinetic-router/src/swap.rs`, line 197

## Recommended Fix

```rust
// Change:
let protocol_fee = k2_shared::utils::percent_mul(to_amount_received, swap_config.flash_loan_premium_bps)?;

// To:
let protocol_fee = k2_shared::utils::percent_mul_up(to_amount_received, swap_config.flash_loan_premium_bps)?;
```
