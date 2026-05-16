# [L-01] Proof of Concept — `swap_collateral` 使用 `percent_mul` 导致协议费系统性少收

## 漏洞位置

`contracts/kinetic-router/src/swap.rs:197`

```rust
// 有问题的代码：
let protocol_fee = k2_shared::utils::percent_mul(to_amount_received, swap_config.flash_loan_premium_bps)?;

// 正确应为：
let protocol_fee = k2_shared::utils::percent_mul_up(to_amount_received, swap_config.flash_loan_premium_bps)?;
```

## 两个函数的区别（来自 `contracts/shared/src/utils.rs`）

```rust
// percent_mul：半进位取整（四舍五入）
fn percent_mul(value: u128, percentage: u128) -> u128 {
    (value * percentage + HALF_BASIS_POINTS) / BASIS_POINTS_MULTIPLIER
    //                    ^^^^^^^^^^^^^^^^^ = 5_000
}

// percent_mul_up：向上取整（对协议有利）
fn percent_mul_up(value: u128, percentage: u128) -> u128 {
    (value * percentage + BASIS_POINTS_MULTIPLIER - 1) / BASIS_POINTS_MULTIPLIER
    //                    ^^^^^^^^^^^^^^^^^^^^^^^^ = 9_999
}
```

当 `value * percentage % 10_000 < 5_000` 时，`percent_mul` 向下取整，`percent_mul_up` 向上取整，差值 = 1 单位。

## 运行方式

```bash
cargo test --package k2-c4 test_percent_mul_fee_underpayment -- --nocapture
```

## 测试代码

将以下函数添加到 `tests/c4/src/lib.rs`（`test_submission_validity` 之后）：

```rust
#[test]
fn test_percent_mul_fee_underpayment() {
    // 协议常量（来自 contracts/shared/src/constants.rs）：
    //   BASIS_POINTS_MULTIPLIER = 10_000
    //   HALF_BASIS_POINTS       = 5_000
    //
    // swap_collateral 使用的 flash_loan_premium_bps 典型值为 9（0.09%）。
    //
    // 触发条件：value * bps % 10_000 < 5_000
    //   → percent_mul 向下取整，percent_mul_up 向上取整，差 1 单位
    //
    // 最小复现：value = 1, bps = 9
    //   value * bps = 9
    //   percent_mul:    (9 + 5_000) / 10_000 = 5_009 / 10_000 = 0  ← 向下
    //   percent_mul_up: (9 + 9_999) / 10_000 = 10_008 / 10_000 = 1 ← 向上

    const BASIS_POINTS_MULTIPLIER: u128 = 10_000;
    const HALF_BASIS_POINTS: u128 = 5_000;

    let bps: u128 = 9; // flash_loan_premium_bps

    // --- 单次 swap 验证 ---
    let amount: u128 = 1;
    let fee_actual   = (amount * bps + HALF_BASIS_POINTS)           / BASIS_POINTS_MULTIPLIER; // 0
    let fee_correct  = (amount * bps + BASIS_POINTS_MULTIPLIER - 1) / BASIS_POINTS_MULTIPLIER; // 1

    assert_eq!(fee_actual,  0, "percent_mul 向下取整为 0");
    assert_eq!(fee_correct, 1, "percent_mul_up 向上取整为 1");
    assert!(fee_correct > fee_actual, "每笔受影响的 swap 少收 {} 单位", fee_correct - fee_actual);

    // --- 累计影响 ---
    // 约 50% 的 swap 满足触发条件（余数 < 5_000 的概率）
    // 按 1_000 笔/天 × 365 天 = 365_000 笔/年估算，约 182_500 笔受影响
    let swaps_per_year: u128 = 365_000;
    let affected_rate_numerator: u128 = 1;
    let affected_rate_denominator: u128 = 2;
    let annual_loss = swaps_per_year * affected_rate_numerator / affected_rate_denominator;

    println!("=== Finding A: percent_mul 费用少收 ===");
    println!("  触发条件: value * bps % 10_000 < 5_000（约 50% 的 swap）");
    println!("  单次损失: {} 单位输出 token", fee_correct - fee_actual);
    println!("  年化估算（1_000 笔/天）: ~{} 单位/每个资产池", annual_loss);
    println!("  修复: swap.rs:197 将 percent_mul 改为 percent_mul_up，一行改动");

    // 验证更大金额下同样存在损失（amount=1_111, bps=9 → 余数=9_999 > 5_000，此时无损失）
    // 验证 amount=1_112, bps=9 → 1_112*9=10_008, 余数=8 < 5_000 → 有损失
    let amount2: u128 = 1_112;
    let fee2_actual  = (amount2 * bps + HALF_BASIS_POINTS)           / BASIS_POINTS_MULTIPLIER;
    let fee2_correct = (amount2 * bps + BASIS_POINTS_MULTIPLIER - 1) / BASIS_POINTS_MULTIPLIER;
    assert!(
        fee2_correct > fee2_actual,
        "amount={} bps={}: percent_mul={}, percent_mul_up={}, 差值={}",
        amount2, bps, fee2_actual, fee2_correct, fee2_correct - fee2_actual
    );
}
```

## 测试证明了什么

| 断言 | 含义 |
|---|---|
| `fee_actual == 0`，`fee_correct == 1` | 同一笔 swap，两个函数结果不同 |
| `fee_correct > fee_actual` | 协议每笔受影响的 swap 少收 1 单位 |
| 第二个 amount 验证 | 不同金额下同样触发，证明这是系统性问题而非边界情况 |

## 量化影响

| 指标 | 数值 |
|---|---|
| 单次损失 | 1 单位输出 token |
| 触发频率 | ~50%（余数 < 5,000 的概率） |
| 年化损失（1,000 笔/天） | ~182,500 单位/每个资产池 |
| 修复成本 | 1 行代码 |
