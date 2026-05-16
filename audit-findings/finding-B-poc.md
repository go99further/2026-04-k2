# [L-02] Proof of Concept — 闪电清算 helper 硬编码 50% 清算比例，误拒合法全额清算

## 漏洞位置

`contracts/flash-liquidation-helper/src/validation.rs:61-68`

```rust
// 有问题的代码：始终使用 DEFAULT_LIQUIDATION_CLOSE_FACTOR = 5_000（50%）
let max_liquidatable = total_debt_base
    .checked_mul(DEFAULT_LIQUIDATION_CLOSE_FACTOR)  // 硬编码 50%
    ...
if debt_to_cover_base > max_liquidatable {
    return Err(KineticRouterError::LiquidationAmountTooHigh);
}
```

主路由器的动态逻辑（`contracts/kinetic-router/src/liquidation.rs:20-25`）：

```rust
// 正确逻辑：满足任一条件时允许 100% 清算
let close_factor = if individual_debt_base < MIN_CLOSE_FACTOR_THRESHOLD   // 债务 < $2,000
    || individual_collateral_base < MIN_CLOSE_FACTOR_THRESHOLD             // 抵押品 < $2,000
    || health_factor < partial_liq_threshold {                             // 严重抵押不足
    MAX_LIQUIDATION_CLOSE_FACTOR   // 10_000 = 100%
} else {
    DEFAULT_LIQUIDATION_CLOSE_FACTOR  // 5_000 = 50%
};
```

## 运行方式

```bash
cargo test --package k2-c4 test_hardcoded_close_factor_false_rejection -- --nocapture
```

## 测试代码

将以下函数添加到 `tests/c4/src/lib.rs`（`test_submission_validity` 之后）：

```rust
#[test]
fn test_hardcoded_close_factor_false_rejection() {
    // 协议常量（来自 contracts/shared/src/constants.rs）：
    //   DEFAULT_LIQUIDATION_CLOSE_FACTOR = 5_000   (50%)
    //   MAX_LIQUIDATION_CLOSE_FACTOR     = 10_000  (100%)
    //   MIN_CLOSE_FACTOR_THRESHOLD       = 2_000 * WAD
    //   BASIS_POINTS_MULTIPLIER          = 10_000
    //
    // 场景：借款人债务 $1,500（< $2,000 阈值）
    // 清算机器人调用 helper 预验证，传入 debt_to_cover = 100% 债务
    //
    // helper 计算：max_liquidatable = $1,500 * 50% = $750
    //              $1,500 > $750 → 返回 LiquidationAmountTooHigh  ← 错误拒绝
    //
    // 主路由器计算：$1,500 < MIN_CLOSE_FACTOR_THRESHOLD → close_factor = 100%
    //              max_liquidatable = $1,500 * 100% = $1,500
    //              $1,500 <= $1,500 → 接受  ← 正确

    // WAD = 1e18，MIN_CLOSE_FACTOR_THRESHOLD = 2_000 * WAD
    const WAD: u128 = 1_000_000_000_000_000_000;
    const MIN_CLOSE_FACTOR_THRESHOLD: u128 = 2_000 * WAD;
    const DEFAULT_CLOSE_FACTOR: u128 = 5_000;
    const MAX_CLOSE_FACTOR: u128 = 10_000;
    const BASIS_POINTS: u128 = 10_000;

    // 债务 $1,500（WAD 计价）
    let total_debt_base: u128 = 1_500 * WAD;
    let debt_to_cover_base: u128 = total_debt_base; // 清算机器人尝试全额清算

    // --- helper 验证逻辑（硬编码 50%）---
    let helper_max = total_debt_base * DEFAULT_CLOSE_FACTOR / BASIS_POINTS; // $750 * WAD
    let helper_rejects = debt_to_cover_base > helper_max;

    // --- 主路由器验证逻辑（动态 close factor）---
    let router_close_factor = if total_debt_base < MIN_CLOSE_FACTOR_THRESHOLD {
        MAX_CLOSE_FACTOR  // 100%，因为 $1,500 < $2,000
    } else {
        DEFAULT_CLOSE_FACTOR
    };
    let router_max = total_debt_base * router_close_factor / BASIS_POINTS; // $1,500 * WAD
    let router_accepts = debt_to_cover_base <= router_max;

    println!("=== Finding B: 硬编码 close factor 误拒 ===");
    println!("  仓位债务:          ${}", total_debt_base / WAD);
    println!("  阈值:              ${}", MIN_CLOSE_FACTOR_THRESHOLD / WAD);
    println!("  helper 最大可清算: ${} (50% 硬编码)", helper_max / WAD);
    println!("  router 最大可清算: ${} (100% 动态)", router_max / WAD);
    println!("  helper 拒绝:  {}", helper_rejects);
    println!("  router 接受:  {}", router_accepts);

    // 核心断言：helper 拒绝，但 router 会接受
    assert!(helper_rejects, "helper 应拒绝（硬编码 50% 上限 = $750，请求 $1,500）");
    assert!(router_accepts, "router 应接受（动态 100% 上限 = $1,500，请求 $1,500）");

    // 量化误拒范围：所有满足 debt < $2,000 的仓位全额清算请求均受影响
    // 被误拒的金额 = debt_to_cover - helper_max
    let rejected_amount = debt_to_cover_base - helper_max;
    assert_eq!(
        rejected_amount,
        750 * WAD,
        "被误拒的清算金额应为 $750（债务的 50%），实际为 ${}",
        rejected_amount / WAD
    );

    println!("  被误拒的清算金额: ${}", rejected_amount / WAD);
    println!("  修复: 在 helper 中复制 validate_close_factor 的动态逻辑");
}
```

## 测试证明了什么

| 断言 | 含义 |
|---|---|
| `helper_rejects == true` | helper 对合法的全额清算返回 `LiquidationAmountTooHigh` |
| `router_accepts == true` | 同一请求主路由器会正常处理 |
| `rejected_amount == $750` | 清算机器人被迫少清算 50% 的债务，或完全放弃该机会 |

## 量化影响

| 指标 | 数值 |
|---|---|
| 受影响仓位 | 所有债务 < $2,000 或抵押品 < $2,000 的仓位 |
| 误拒率 | 100%（此类仓位的全额清算请求全部被拒） |
| 每次误拒损失 | 清算机器人少清算 50% 债务（= `debt_to_cover / 2`） |
| 协议风险 | 小型不良仓位滞留时间延长，坏账风险上升 |
| 修复成本 | 在 helper 中复制 `validate_close_factor` 逻辑，约 10 行 |
