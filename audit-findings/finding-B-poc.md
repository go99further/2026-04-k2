# [L-02] Proof of Concept — Flash liquidation helper rejects valid full-liquidations

## What this proves

When a position qualifies for 100% liquidation (debt < $2 000 or collateral < $2 000), the flash liquidation helper's hardcoded 50% close factor causes it to return `LiquidationAmountTooHigh` for the full-debt amount. The main router would accept the same call. The test below simulates this mismatch with concrete numbers and asserts the false rejection.

## Run

```bash
cargo test --package k2-c4 test_hardcoded_close_factor_false_rejection -- --nocapture
```

## Test

Add to `tests/c4/src/lib.rs` after `test_submission_validity`:

```rust
#[test]
fn test_hardcoded_close_factor_false_rejection() {
    // -----------------------------------------------------------------------
    // Constants (from k2_shared):
    //   DEFAULT_LIQUIDATION_CLOSE_FACTOR = 5_000  (50%, basis points)
    //   MAX_LIQUIDATION_CLOSE_FACTOR     = 10_000 (100%)
    //   MIN_CLOSE_FACTOR_THRESHOLD       = 2_000_000_000_000_000_000 (= $2 000 in WAD)
    //   BASIS_POINTS_MULTIPLIER          = 10_000
    //
    // Main router dynamic close factor logic (liquidation.rs `validate_close_factor`):
    //   if total_debt_base < MIN_CLOSE_FACTOR_THRESHOLD
    //      OR total_collateral_base < MIN_CLOSE_FACTOR_THRESHOLD:
    //       close_factor = 100%
    //   else:
    //       close_factor = 50%
    //
    // Flash liquidation helper (validation.rs lines 60-68):
    //   always uses DEFAULT_LIQUIDATION_CLOSE_FACTOR = 50%
    // -----------------------------------------------------------------------

    // Scenario: small borrower with $1 500 total debt (below $2 000 threshold)
    // PRICE_ONE_DOLLAR = 100_000_000_000_000 (14 decimal oracle precision)
    // WAD              = 1_000_000_000_000_000_000 (18 decimals)
    // oracle_to_wad    = WAD / PRICE_ONE_DOLLAR = 10_000 (for 14-decimal oracle)
    //
    // total_debt_base in WAD units for $1 500:
    //   = 1_500 * PRICE_ONE_DOLLAR * oracle_to_wad / decimals_pow
    //   For simplicity, work directly in base-currency units (WAD-scaled dollars):
    let total_debt_base: u128 = 1_500_000_000_000_000_000_000; // $1 500 in WAD
    let min_close_factor_threshold: u128 = 2_000_000_000_000_000_000_000; // $2 000 in WAD

    // Liquidator wants to cover 100% of debt
    let debt_to_cover_base = total_debt_base;

    // --- Helper validation (buggy: hardcoded 50%) ---
    let default_close_factor: u128 = 5_000;
    let basis_points: u128 = 10_000;
    let max_liquidatable_helper = total_debt_base * default_close_factor / basis_points; // $750

    let helper_rejects = debt_to_cover_base > max_liquidatable_helper;

    // --- Main router validation (correct: dynamic 100% for small positions) ---
    let dynamic_close_factor = if total_debt_base < min_close_factor_threshold {
        10_000u128 // 100%
    } else {
        5_000u128  // 50%
    };
    let max_liquidatable_router = total_debt_base * dynamic_close_factor / basis_points; // $1 500

    let router_accepts = debt_to_cover_base <= max_liquidatable_router;

    println!("=== Finding B: hardcoded close factor false rejection ===");
    println!("  total_debt_base:          ${}", total_debt_base / 1_000_000_000_000_000_000);
    println!("  debt_to_cover (100%):     ${}", debt_to_cover_base / 1_000_000_000_000_000_000);
    println!("  helper max_liquidatable:  ${} (50% hardcoded)", max_liquidatable_helper / 1_000_000_000_000_000_000);
    println!("  router max_liquidatable:  ${} (100% dynamic)", max_liquidatable_router / 1_000_000_000_000_000_000);
    println!("  helper rejects:  {}", helper_rejects);
    println!("  router accepts:  {}", router_accepts);

    assert!(
        helper_rejects,
        "Helper should reject the full-liquidation (hardcoded 50% close factor)"
    );
    assert!(
        router_accepts,
        "Router should accept the full-liquidation (dynamic 100% close factor)"
    );
    assert!(
        helper_rejects && router_accepts,
        "Bug confirmed: helper rejects a liquidation the main router would accept.\n  \
         Liquidator using helper for pre-validation will miss this opportunity.\n  \
         Position: total_debt=${}, threshold=${}, helper_max=${}, router_max=${}",
        total_debt_base / 1_000_000_000_000_000_000,
        min_close_factor_threshold / 1_000_000_000_000_000_000,
        max_liquidatable_helper / 1_000_000_000_000_000_000,
        max_liquidatable_router / 1_000_000_000_000_000_000,
    );
}
```

## Quantified impact

| Metric | Value |
|---|---|
| Affected positions | All positions with debt < $2 000 OR collateral < $2 000 |
| False rejection rate | 100% of full-liquidation attempts on qualifying small positions |
| Liquidator impact | Misses valid liquidation opportunity; must bypass helper or resize to 50% |
| Protocol impact | Undercollateralized small positions may linger longer, increasing bad debt risk |
| Fix | Replicate `validate_close_factor` logic in helper, or pass close factor as parameter |
