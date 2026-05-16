# [M-01] Bad debt deficit is double-subtracted in `collect_protocol_reserves` and `get_protocol_reserves`, permanently locking protocol fees

## Summary

When bad debt is socialized, the protocol burns the borrower's remaining debt tokens (`total_borrow -= D`) and records `reserve_deficit += D`. The `collect_protocol_reserves` function (and the `get_protocol_reserves` view) then compute `available_reserves = underlying_balance - (total_supply - total_borrow)`, which already reflects the bad debt loss (because `total_borrow` was reduced). They then subtract `deficit` a second time, producing a result that is `D` lower than the true collectible amount. This permanently prevents the admin from collecting legitimate protocol fees equal to the deficit amount.

## Vulnerability Detail

### How protocol reserves are computed

```
available_reserves = underlying_balance - (total_supply - total_borrow)
```

This formula captures the "extra" tokens in the aToken contract beyond what depositors can claim net of borrower obligations.

### What happens during bad debt socialization

When a borrower's collateral is fully seized and remaining debt `D` is burned:

1. `total_borrow -= D` (debt tokens burned — borrower's obligation erased)
2. `reserve_deficit += D` (deficit recorded for external coverage)
3. `underlying_balance` is **unchanged** (the tokens were already borrowed out and never returned)

After this, `available_reserves` becomes:

```
available_reserves_after = underlying_balance - (total_supply - (total_borrow - D))
                         = underlying_balance - total_supply + total_borrow - D
                         = available_reserves_before - D
```

The bad debt loss is **already reflected** in `available_reserves` through the reduced `total_borrow`.

### The double subtraction

`contracts/kinetic-router/src/treasury.rs:55-56`:
```rust
let deficit = storage::get_reserve_deficit(&env, &asset);
let collectible_reserves = available_reserves.saturating_sub(deficit);
```

`contracts/kinetic-router/src/calculation.rs:1267-1268`:
```rust
let deficit = crate::storage::get_reserve_deficit(env, asset);
Ok(raw_reserves.saturating_sub(deficit))
```

Both functions subtract `deficit` from a value that already accounts for the bad debt. The result is:

```
collectible = (available_reserves_before - D) - D = available_reserves_before - 2D
```

### Concrete example

| State | underlying_balance | total_supply | total_borrow | available_reserves | deficit | collectible |
|---|---|---|---|---|---|---|
| Before bad debt | 1000 | 900 | 100 | 200 | 0 | 200 (correct) |
| After bad debt D=50 | 1000 | 900 | 50 | 150 | 50 | **100** (wrong, should be 150) |

The admin loses access to 50 tokens of legitimate protocol fees. Those tokens remain in the aToken contract but cannot be collected.

## Impact

- **Protocol fee collection is permanently impaired** by the deficit amount after any bad debt event. The admin cannot collect `D` tokens of legitimate reserves without first externally covering the deficit via `cover_deficit`.
- **`get_protocol_reserves` returns an understated value** to off-chain tooling and governance, misrepresenting the protocol's financial health.
- The locked amount scales with the total bad debt across all reserves. In a protocol with significant bad debt history, this could represent a material sum.

## Code Location

- `contracts/kinetic-router/src/treasury.rs`, lines 55–56
- `contracts/kinetic-router/src/calculation.rs`, lines 1267–1268

## Proof of Concept

```rust
// Scenario: protocol has 200 in reserves, bad debt D=50 occurs
// After bad debt:
//   underlying_balance = 1000
//   total_supply       = 900
//   total_borrow       = 50   (was 100, burned 50)
//   reserve_deficit    = 50

// available_liquidity = 900 - 50 = 850
// available_reserves  = 1000 - 850 = 150  ← already reduced by D

// Code then does:
// collectible = 150 - 50 = 100  ← subtracts D again

// Admin can only collect 100, not 150.
// 50 tokens are permanently locked until deficit is externally covered.
```

## Recommended Fix

Remove the `saturating_sub(deficit)` from both functions. The `available_reserves` calculation already accounts for bad debt through the reduced `total_borrow`. The deficit variable is for tracking external coverage obligations, not for adjusting the reserve calculation.

**`treasury.rs`** — remove lines 54–56:
```rust
// DELETE:
let deficit = storage::get_reserve_deficit(&env, &asset);
let collectible_reserves = available_reserves.saturating_sub(deficit);

// REPLACE with:
let collectible_reserves = available_reserves;
```

**`calculation.rs`** — remove lines 1266–1268:
```rust
// DELETE:
let deficit = crate::storage::get_reserve_deficit(env, asset);
Ok(raw_reserves.saturating_sub(deficit))

// REPLACE with:
Ok(raw_reserves)
```

If the intent is to prevent collecting reserves while bad debt exists (a policy decision), the correct approach is to check `if deficit > 0 { return Ok(0); }` rather than subtracting — but this is a separate design choice from the accounting correctness issue.
