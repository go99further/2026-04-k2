# K2 借贷协议 — 安全审计发现

对 [K2 借贷协议](https://github.com/code-423n4/2026-04-k2)（Soroban/Rust，Stellar 链）进行独立安全审计，提交至 [Code4rena 竞赛](https://code4rena.com/audits/2026-04-k2)（奖池：$135,000 USDC）。

## 发现汇总

| 编号 | 标题 | 严重性 | 量化影响 |
|---|---|---|---|
| [M-01](finding-C-deficit-double-counting.md) | `collect_protocol_reserves` 中坏账亏空被二次扣减 | 中危 | 协议费用被永久锁定量 = 2 × 坏账金额；管理员须注入等额真实资金才能解锁本已赚取的费用 |
| [L-01](finding-A-percent-mul-swap.md) | `swap_collateral` 使用 `percent_mul`（向下取整）而非 `percent_mul_up` 计算协议费 | 低危 | 每笔受影响的 swap 少收 1 单位；约 50% 的 swap 受影响，按 1,000 笔/天估算，每个资产池每年系统性少收约 365,000 单位 |
| [L-02](finding-B-hardcoded-close-factor.md) | 闪电清算 helper 硬编码 50% 清算比例，错误拒绝合法的全额清算 | 低危 | 对债务或抵押品 < $2,000 的仓位，全额清算请求 100% 被误拒；主路由器会正常接受同一请求 |

## M-01 深度分析：坏账二次扣减

影响最大的发现。坏账 `D` 被社会化时：

1. `total_borrow -= D` — `raw_reserves` 已经因此减少 `D`
2. 代码随后再次减去 `deficit`（= `D`）

**结果：** `可提取储备 = raw_reserves_before − 2D`，而正确值应为 `raw_reserves_before − D`

具体示例（`D = 50` 个代币）：

| 状态 | raw_reserves | deficit | get_protocol_reserves 返回值 | 是否正确 |
|---|---|---|---|---|
| 坏账发生前 | 200 | 0 | 200 | ✓ |
| 坏账发生后 | 150 | 50 | **100** | ✗（应为 150） |

管理员被锁在 50 个代币的合法费用之外，必须调用 `cover_deficit(50)` 注入真实资金才能解锁。**锁定金额随协议历史累计坏账线性增长。**

**修复：** 删除 `treasury.rs:55` 和 `calculation.rs:1267` 各一行的 `saturating_sub(deficit)` 即可。

## PoC 测试

每个发现均附有可运行的 Rust PoC，使用协议自带测试脚手架（`tests/c4`）：

| 发现 | PoC 文件 | 核心断言 |
|---|---|---|
| M-01 | [finding-C-poc.md](finding-C-poc.md) | `reserves_before − reserves_after == 2 × deficit` |
| L-01 | [finding-A-poc.md](finding-A-poc.md) | `percent_mul(1, 9) == 0`，而 `percent_mul_up(1, 9) == 1` |
| L-02 | [finding-B-poc.md](finding-B-poc.md) | helper 拒绝 `debt_to_cover=$1,500`（超 50% 上限）；router 接受（动态 100% 上限） |

## 技术栈

- **协议：** Stellar 链 Soroban 智能合约（Rust）
- **测试框架：** `soroban-sdk`，`mock_all_auths`，WASM 合约导入
- **审计范围：** Kinetic Router、aToken、Debt Token、闪电清算 Helper、价格预言机

---

## 简历参考描述

> **K2 借贷协议安全审计** · Code4rena（奖池 $135,000 USDC）
> 对 Stellar 链 Soroban/Rust 借贷协议进行独立安全审计，发现 3 个漏洞。核心发现（M-01）：储备金计算中坏账亏空被二次扣减，导致协议合法费用被永久锁定，锁定量等于累计坏账总额，管理员须注入等额真实资金方可解锁。为全部发现提供可运行的 Rust PoC 测试。
