# K2 Lending Protocol — Audit Findings

Independent security audit of the [K2 lending protocol](https://github.com/code-423n4/2026-04-k2) (Soroban/Rust, Stellar), submitted to the [Code4rena contest](https://code4rena.com/audits/2026-04-k2) (prize pool: $135,000 USDC).

## Findings Summary

| ID | Title | Severity | Quantified Impact |
|---|---|---|---|
| [M-01](finding-C-deficit-double-counting.md) | Bad debt deficit double-subtracted in `collect_protocol_reserves` | Medium | Protocol fees locked = 2× bad debt amount; admin must inject real capital to unlock fees the protocol legitimately earned |
| [L-01](finding-A-percent-mul-swap.md) | `swap_collateral` uses `percent_mul` instead of `percent_mul_up` for protocol fee | Low | 1 unit underpayment per affected swap; ~50% of swaps affected → ~365,000 units/year per reserve at 1,000 swaps/day |
| [L-02](finding-B-hardcoded-close-factor.md) | Flash liquidation helper hardcodes 50% close factor, rejecting valid full-liquidations | Low | 100% false-rejection rate for full-liquidation attempts on positions with debt or collateral < $2,000 |

## Proof of Concept Tests

Each finding includes a runnable Rust PoC using the protocol's own test scaffold (`tests/c4`):

| Finding | PoC | Key Assertion |
|---|---|---|
| M-01 | [finding-C-poc.md](finding-C-poc.md) | `reserves_before − reserves_after == 2 × deficit` |
| L-01 | [finding-A-poc.md](finding-A-poc.md) | `percent_mul(1, 9) == 0` while `percent_mul_up(1, 9) == 1` |
| L-02 | [finding-B-poc.md](finding-B-poc.md) | Helper rejects `debt_to_cover=$1,500` (> 50% cap); router accepts (100% dynamic cap) |

## M-01 Deep Dive: Bad Debt Double-Subtraction

The most impactful finding. When bad debt `D` is socialized:

1. `total_borrow -= D` — already reduces `raw_reserves` by `D`
2. Code then subtracts `deficit` (= `D`) a second time

**Result:** `collectible = raw_reserves_before − 2D` instead of `raw_reserves_before − D`

Concrete example with `D = 50` tokens:

| State | raw_reserves | deficit | get_protocol_reserves | Correct? |
|---|---|---|---|---|
| Before bad debt | 200 | 0 | 200 | ✓ |
| After bad debt | 150 | 50 | **100** | ✗ (should be 150) |

The admin is locked out of 50 tokens of legitimate fees and must call `cover_deficit(50)` — spending real capital — to unlock fees the protocol already earned. The locked amount scales linearly with total historical bad debt across all reserves.

**Fix:** Remove `saturating_sub(deficit)` from `treasury.rs:55` and `calculation.rs:1267` — one line each.

## Technical Stack

- **Protocol:** Soroban smart contracts (Rust) on Stellar
- **Test framework:** `soroban-sdk` with `mock_all_auths`, WASM contract imports
- **Audit scope:** Kinetic Router, aToken, Debt Token, Flash Liquidation Helper, Price Oracle
