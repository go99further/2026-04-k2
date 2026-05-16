# [M-01] Proof of Concept — Bad debt deficit double-subtracted

## Setup

Uses the `Setup` scaffold from `tests/c4/src/lib.rs`. No modifications to that file are needed.

Add the test function below to `tests/c4/src/lib.rs` (after the `test_submission_validity` function).

Run with:

```bash
cargo test --package k2-c4 test_deficit_double_subtraction -- --nocapture
```

---

## Test

```rust
#[test]
fn test_deficit_double_subtraction() {
    let env = Env::default();
    let setup = Setup::new(&env); // timestamp = 1000, sequence = 100

    // -----------------------------------------------------------------------
    // Step 1: Create high utilization so protocol reserves accumulate.
    //
    // big_borrower supplies 100 M asset_a and borrows 80 M asset_b.
    // This puts the pool at ~80% utilization (= optimal), which drives the
    // borrow rate to slope1 = 40% APY.  After one day the reserve factor
    // (10%) captures ~8 700 tokens of asset_b as protocol reserves.
    // -----------------------------------------------------------------------
    let big_borrower = Address::generate(&env);
    let large_amount: i128 = LP_SEED; // 100 M tokens (7 decimals)

    setup.asset_a_mint.mint(&big_borrower, &large_amount);
    setup.asset_a_token.approve(
        &big_borrower,
        &setup.router_addr,
        &large_amount,
        &(env.ledger().sequence() + 100_000),
    );

    setup.router.supply(
        &big_borrower,
        &setup.asset_a,
        &(large_amount as u128),
        &big_borrower,
        &0u32,
    );
    setup.router.set_user_use_reserve_as_coll(&big_borrower, &setup.asset_a, &true);

    let big_borrow: u128 = 800_000_000_000_000; // 80 M tokens
    setup.router.borrow(
        &big_borrower,
        &setup.asset_b,
        &big_borrow,
        &1u32,
        &0u32,
        &big_borrower,
    );

    // -----------------------------------------------------------------------
    // Step 2: Create a small position that will become bad debt.
    //
    // small_borrower supplies 125 tokens of asset_a (LTV 80% → can borrow
    // 100 tokens of asset_b).  After the price crash the collateral is worth
    // ~$0.125 against ~$100 of debt, so the entire position becomes bad debt.
    // -----------------------------------------------------------------------
    let small_borrower = Address::generate(&env);
    let small_collateral: i128 = 1_250_000_000; // 125 tokens

    setup.asset_a_mint.mint(&small_borrower, &small_collateral);
    setup.asset_a_token.approve(
        &small_borrower,
        &setup.router_addr,
        &small_collateral,
        &(env.ledger().sequence() + 100_000),
    );

    setup.router.supply(
        &small_borrower,
        &setup.asset_a,
        &(small_collateral as u128),
        &small_borrower,
        &0u32,
    );
    setup.router.set_user_use_reserve_as_coll(&small_borrower, &setup.asset_a, &true);

    let small_borrow: u128 = 1_000_000_000; // 100 tokens
    setup.router.borrow(
        &small_borrower,
        &setup.asset_b,
        &small_borrow,
        &1u32,
        &0u32,
        &small_borrower,
    );

    // -----------------------------------------------------------------------
    // Step 3: Advance time by one day so interest accrues.
    // -----------------------------------------------------------------------
    env.ledger().set(soroban_sdk::testutils::LedgerInfo {
        sequence_number: 200,
        protocol_version: 23,
        timestamp: 1000 + 86_400,
        network_id: Default::default(),
        base_reserve: 10,
        min_temp_entry_ttl: 10,
        min_persistent_entry_ttl: 10,
        max_entry_ttl: 1_000_000,
    });

    // -----------------------------------------------------------------------
    // Step 4: Snapshot protocol reserves BEFORE bad debt.
    //
    // get_protocol_reserves calls update_state internally (interest accrual
    // only — no oracle prices needed).
    // -----------------------------------------------------------------------
    let reserves_before = setup
        .router
        .get_protocol_reserves(&setup.asset_b)
        .expect("get_protocol_reserves failed");

    assert!(
        reserves_before > 0,
        "Protocol should have accumulated reserves after one day of 80% utilization. Got: {}",
        reserves_before
    );

    // -----------------------------------------------------------------------
    // Step 5: Refresh oracle prices at the new timestamp.
    //
    // The original overrides were set at timestamp 1000; after advancing one
    // day the router's default 3 600-second staleness window rejects them.
    // Reset the circuit breaker for each asset and publish fresh prices:
    //   asset_b stays at $1.00
    //   asset_a crashes to $0.001 (99.9% drop)
    // -----------------------------------------------------------------------
    let new_expiry = env.ledger().timestamp() + 604_800; // 7 days from now

    let asset_a_enum = price_oracle::Asset::Stellar(setup.asset_a.clone());
    let asset_b_enum = price_oracle::Asset::Stellar(setup.asset_b.clone());

    // Refresh asset_b at $1.00
    setup.oracle.reset_circuit_breaker(&setup.admin, &asset_b_enum);
    setup.oracle.set_manual_override(
        &setup.admin,
        &asset_b_enum,
        &Some(PRICE_ONE_DOLLAR),
        &Some(new_expiry),
    );

    // Crash asset_a to $0.001
    // PRICE_ONE_DOLLAR = 100_000_000_000_000 (14 decimals)
    // $0.001 = 100_000_000_000 (14 decimals)
    let crashed_price: u128 = 100_000_000_000;
    setup.oracle.reset_circuit_breaker(&setup.admin, &asset_a_enum);
    setup.oracle.set_manual_override(
        &setup.admin,
        &asset_a_enum,
        &Some(crashed_price),
        &Some(new_expiry),
    );

    // -----------------------------------------------------------------------
    // Step 6: Liquidate small_borrower.
    //
    // With asset_a at $0.001:
    //   collateral value  = 125 tokens × $0.001 = $0.125
    //   debt value        ≈ 100 tokens × $1.00  = $100
    //
    // collateral_cap_triggered = true because the collateral needed to cover
    // the full debt (≈105 000 tokens) far exceeds the 125 tokens available.
    //
    // The liquidator covers only the tiny slice of debt backed by collateral
    // (~0.12 tokens); the remaining ~99.88 tokens become bad debt (deficit D).
    // -----------------------------------------------------------------------
    let liquidator = Address::generate(&env);
    let liquidator_funds: i128 = 2_000_000_000; // 200 tokens — more than enough
    setup.asset_b_mint.mint(&liquidator, &liquidator_funds);
    setup.asset_b_token.approve(
        &liquidator,
        &setup.router_addr,
        &liquidator_funds,
        &(env.ledger().sequence() + 100_000),
    );

    setup.router.liquidation_call(
        &liquidator,
        &setup.asset_a,   // collateral asset
        &setup.asset_b,   // debt asset
        &small_borrower,
        &small_borrow,    // attempt to cover full debt; router will cap at collateral value
        &false,           // receive underlying, not aToken
    );

    // -----------------------------------------------------------------------
    // Step 7: Read post-bad-debt state.
    // -----------------------------------------------------------------------
    let reserves_after = setup
        .router
        .get_protocol_reserves(&setup.asset_b)
        .expect("get_protocol_reserves failed after bad debt");

    let deficit = setup.router.get_reserve_deficit(&setup.asset_b);

    assert!(
        deficit > 0,
        "Bad debt should have been recorded as a deficit. Got: {}",
        deficit
    );

    // -----------------------------------------------------------------------
    // Step 8: Assert the double-subtraction bug.
    //
    // Correct accounting:
    //   raw_reserves_after = raw_reserves_before − D   (total_borrow reduced)
    //   get_protocol_reserves should return raw_reserves_after = reserves_before − D
    //   → reserves_before − reserves_after == D == deficit
    //
    // Buggy accounting (what the code actually does):
    //   get_protocol_reserves returns raw_reserves_after − deficit
    //                                = (reserves_before − D) − D
    //                                = reserves_before − 2D
    //   → reserves_before − reserves_after == 2 × deficit
    //
    // The assertion below PASSES, proving the bug is present.
    // -----------------------------------------------------------------------
    assert_eq!(
        reserves_before.saturating_sub(reserves_after),
        2 * deficit,
        "Bug confirmed: deficit was subtracted twice.\n  \
         reserves_before = {}\n  \
         reserves_after  = {}\n  \
         deficit         = {}\n  \
         drop            = {} (should be {}, is {})",
        reserves_before,
        reserves_after,
        deficit,
        reserves_before.saturating_sub(reserves_after),
        deficit,
        2 * deficit,
    );

    // Complementary check: the admin is locked out of `deficit` tokens of
    // legitimate fees.  After cover_deficit the view should restore to
    // reserves_before − deficit (one subtraction, not two).
    let cover_caller = Address::generate(&env);
    setup.asset_b_mint.mint(&cover_caller, &(deficit as i128));
    setup.asset_b_token.approve(
        &cover_caller,
        &setup.router_addr,
        &(deficit as i128),
        &(env.ledger().sequence() + 100_000),
    );
    setup
        .router
        .cover_deficit(&cover_caller, &setup.asset_b, &deficit)
        .expect("cover_deficit failed");

    let reserves_after_cover = setup
        .router
        .get_protocol_reserves(&setup.asset_b)
        .expect("get_protocol_reserves failed after cover");

    // After covering the deficit, the view should now return reserves_before − deficit.
    // This confirms that the locked amount equals exactly one deficit, not zero.
    assert_eq!(
        reserves_after_cover,
        reserves_before.saturating_sub(deficit),
        "After cover_deficit, reserves should equal reserves_before − deficit.\n  \
         Expected: {}\n  Got: {}",
        reserves_before.saturating_sub(deficit),
        reserves_after_cover,
    );
}
```

---

## What the test proves

| Assertion | Meaning |
|---|---|
| `reserves_before − reserves_after == 2 × deficit` | The deficit was subtracted twice from `raw_reserves`, not once |
| `reserves_after_cover == reserves_before − deficit` | Covering the deficit "unlocks" the hidden fees, confirming they were never actually lost — only miscounted |

The second assertion also shows that the admin must spend `deficit` tokens of real capital (via `cover_deficit`) to access fees the protocol legitimately earned, even though those fees were never lost from the contract.
