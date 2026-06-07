# Adaptive LP Vault

> 一个 contracts-only 的自适应流动性 Vault 项目。用户存入一对 ERC-20 资产，Vault 铸造份额，并通过 Adapter 管理 Uniswap V2 / Uniswap V3 流动性仓位；合约使用 TWAP 与波动率信号决定是否允许再平衡。
>
> 本项目定位为工程化 DeFi 合约项目：范围保持可实现，测试和安全边界必须完整；v1 以测试网交付和主网 fork 验证为目标。

[![Solidity](https://img.shields.io/badge/Solidity-^0.8.20-blue)](https://soliditylang.org/)
[![Foundry](https://img.shields.io/badge/Built%20with-Foundry-FFDB1C.svg)](https://getfoundry.sh/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 目录

- [项目概述](#项目概述)
- [目标用户与交互流程](#目标用户与交互流程)
- [术语表](#术语表)
- [范围声明](#范围声明)
- [合约架构](#合约架构)
- [合约结构](#合约结构)
- [核心模块 PRD](#核心模块-prd)
- [资金流与会计模型](#资金流与会计模型)
- [再平衡策略](#再平衡策略)
- [权限与角色](#权限与角色)
- [事件设计](#事件设计)
- [错误与 Revert 策略](#错误与-revert-策略)
- [安全模型](#安全模型)
- [测试计划](#测试计划)
- [部署指南](#部署指南)
- [升级与迁移策略](#升级与迁移策略)
- [数值精度与舍入](#数值精度与舍入)
- [验收标准](#验收标准)
- [交付清单](#交付清单)
- [项目亮点](#项目亮点)
- [许可证](#许可证)

---

## 项目概述

Adaptive LP Vault 是一个用于管理 AMM 流动性仓位的智能合约系统。

用户将两种资产，例如 WETH 和 USDC，存入 Vault。Vault 根据用户存入资产的价值铸造 Vault shares。Vault 可以将资产放入不同流动性场所：

- IDLE：资产留在 Vault 中，不提供流动性。
- Uniswap V2：向指定 V2 Pair 提供全区间流动性。
- Uniswap V3：向指定 V3 Pool 的某个 tick range 提供集中流动性。

Vault 使用 TWAP Oracle 读取价格和波动率信息。任何人都可以调用 `canRebalance()` 查询是否满足再平衡条件；满足条件时，任何人都可以调用 `rebalance()` 将仓位从一个场所迁移到另一个场所，但目标、Adapter、收款地址、滑点和价值损失全部由 Vault 校验。

本项目重点展示：

- 双资产 Vault 份额会计。
- AMM 流动性添加、移除和费用收集。
- Adapter Pattern。
- TWAP Oracle 集成。
- 再平衡条件判断。
- Slippage、Reentrancy、Access Control、安全边界。
- Foundry 单元测试、集成测试和主网 fork 测试。

---

## 目标用户与交互流程

### 用户角色

| 用户 | 目标 | 使用功能 |
|---|---|---|
| Depositor | 想获得被动 LP 暴露，不想手动管理仓位 | `deposit`、`withdraw`、`previewDeposit`、`previewWithdraw` |
| Rebalancer | 根据公开条件触发 Vault 再平衡 | `canRebalance`、`rebalance` |
| Admin | 配置策略参数、Adapter、Oracle 和安全阈值 | 参数管理、暂停、恢复 |
| Auditor / Reviewer | 检查合约设计和安全边界 | README、NatSpec、测试、SECURITY.md |

### 用户流程

```text
Depositor
  -> approve token0/token1
  -> deposit(amount0, amount1, minShares, deadline)
  -> receives Vault shares
  -> waits while Vault earns fees or changes value
  -> withdraw(shares, minAmount0, minAmount1, deadline)
  -> receives token0/token1

Rebalancer
  -> canRebalance(targetVenue, params)
  -> if true: rebalance(targetVenue, params)
  -> Vault removes current liquidity
  -> Vault collects fees
  -> Vault adds liquidity to target venue

Admin
  -> configure adapters/oracles/thresholds
  -> pause rebalance or deposit in emergency
  -> cannot directly transfer user assets
```

---

## 术语表

| 术语 | 含义 |
|---|---|
| Vault | 接收用户资产、铸造份额、管理 LP 仓位的主合约 |
| Vault shares | Vault 份额，ERC-20 形式，代表用户对 Vault 资产的比例所有权 |
| token0 / token1 | Vault 支持的一对 ERC-20 资产，例如 WETH / USDC |
| Quote value | 用 `VALUE_SCALE = 1e18` 归一化后的 token1 计价价值；仅用于 shares 会计和 slippage 检查 |
| Venue | 流动性场所，例如 IDLE、Uniswap V2、Uniswap V3 0.05% |
| Adapter | 将不同 AMM 场所抽象成统一接口的合约 |
| TWAP | Time-Weighted Average Price，时间加权平均价格 |
| Rebalance | 移除旧仓位并建立新仓位的操作 |
| Slippage | 实际成交结果相对预期结果的偏差 |
| Idle balance | Vault 中未投入 AMM 的 token0/token1 余额 |
| Position value | Vault 当前仓位按 TWAP 估算后的总价值 |
| Principal | 用户通过 deposit 进入 Vault 的本金资产 |
| Fees | AMM 仓位产生并被 collect 回 Vault 的 token0/token1 |
| BPS | basis points，`10000 = 100%` |

---

## 范围声明

### v1 交付范围

本仓库只交付智能合约、测试、部署脚本和文档。

| 模块 | v1 是否交付 | 说明 |
|---|---|---|
| AdaptiveLPVault | 是 | 主 Vault，负责 deposit、withdraw、shares 和 rebalance 调度 |
| TWAPOracle | 是 | 读取 Uniswap V3 TWAP，用于报价和波动率信号 |
| ILiquidityAdapter | 是 | Adapter 标准接口 |
| UniswapV2Adapter | 是 | 向 V2 Pair add/remove liquidity |
| UniswapV3Adapter | 是 | 至少支持一个 V3 fee tier 的 mint/decrease/collect |
| VaultMath | 是 | 份额、价值、比例、舍入计算 |
| Foundry tests | 是 | 单元测试、集成测试、fork 测试 |
| 前端 | 否 | 不交付 |
| Subgraph / indexer | 否 | 不交付 |
| 真实收益策略优化器 | 否 | v1 只做规则型再平衡，不承诺最优收益 |

### 非目标

- 不实现自动收益承诺。
- 不实现跨链。
- 不支持任意 token。
- 不支持 fee-on-transfer、rebasing、ERC777 callback token。
- 不支持单边入金自动 swap 配平。
- 不支持借贷、杠杆或清算逻辑。
- 不将 TWAP 描述为绝对防操纵价格源。
- 不提供真实主网上线承诺，除非通过审计和额外风险评估。

### v1 关键设计决策

| 决策 | 规则 |
|---|---|
| 入金方式 | v1 只支持双资产入金，`amount0 > 0 && amount1 > 0` |
| 入金比例 | 用户入金比例必须接近 Vault 当前目标比例，超过 `depositRatioToleranceBps` 时 revert |
| 单边入金 | 不支持。单边入金需要 swap 路由、滑点、MEV 和税费 token 处理，v1 不纳入 |
| 再平衡执行 | v1 默认 permissionless，任何人可触发，但目标、参数和滑点全部由合约校验 |
| Rebalancer 奖励 | v1 不内置奖励，避免引入刷 rebalance 的激励攻击 |
| Adapter 管理 | Adapter 必须由 Owner / multisig 配置，用户和 rebalancer 不能指定任意 Adapter |
| 支持资产 | `token0/token1` 部署后不可变；仅支持已在测试中覆盖的标准 ERC-20 |
| 交付级别 | v1 目标是测试网部署、主网 fork 测试和完整文档，不承诺真实主网上线 |

---

## 合约架构

```text
Users
  -> AdaptiveLPVault
      -> VaultMath
      -> TWAPOracle
      -> ILiquidityAdapter
          -> UniswapV2Adapter
          -> UniswapV3Adapter
      -> token0 / token1

Admin / multisig
  -> configure adapters
  -> configure risk params
  -> pause / unpause limited actions

Rebalancer / anyone
  -> canRebalance()
  -> rebalance()
```

设计原则：

- Vault 只保存用户资产和份额会计。
- Adapter 只负责和外部 AMM 交互。
- Oracle 只负责价格和波动率读取。
- Admin 只能配置参数，不提供直接转移用户资产的函数；Adapter / Oracle 变更必须发出事件并在测试中覆盖权限边界。
- Rebalancer 不能指定任意 token、任意路由或任意接收方。

---

## 合约结构

```text
src/
├── AdaptiveLPVault.sol
├── TWAPOracle.sol
├── interfaces/
│   ├── ILiquidityAdapter.sol
│   ├── ITWAPOracle.sol
│   └── IAdaptiveLPVault.sol
├── adapters/
│   ├── UniswapV2Adapter.sol
│   └── UniswapV3Adapter.sol
├── libraries/
│   ├── VaultMath.sol
│   └── TickMathHelper.sol
├── mocks/
│   ├── MockERC20.sol
│   ├── MockV2Pair.sol
│   └── MockV3Pool.sol
script/
├── Deploy.s.sol
└── DeployTestnet.s.sol
test/
├── unit/
│   ├── VaultDepositWithdraw.t.sol
│   ├── VaultAccounting.t.sol
│   ├── TWAPOracle.t.sol
│   ├── V2Adapter.t.sol
│   └── V3Adapter.t.sol
├── integration/
│   └── RebalanceFlow.t.sol
├── fork/
│   └── MainnetFork.t.sol
└── invariant/
    └── VaultInvariant.t.sol
README.md
SECURITY.md
foundry.toml
```

---

## 核心模块 PRD

### AdaptiveLPVault.sol

主 Vault 合约，负责接收 token0/token1、铸造 Vault shares、处理提款和调度再平衡。

> 注意：v1 是 ERC-20 shares 的双资产 Vault。它可以参考 ERC-4626 的 UX 和命名，但不声明自己是严格 ERC-4626，因为 ERC-4626 标准是单资产 Vault。

#### 核心状态

| 状态 | 含义 |
|---|---|
| `token0` | Vault 支持的第一个资产 |
| `token1` | Vault 支持的第二个资产 |
| `currentVenue` | 当前流动性场所 |
| `currentAdapter` | 当前 Adapter |
| `adapterOf[venue]` | 指定 Venue 对应的白名单 Adapter |
| `idle0()` | Vault 当前持有且未投入 AMM 的 token0 实时余额 |
| `idle1()` | Vault 当前持有且未投入 AMM 的 token1 实时余额 |
| `lastRebalanceAt` | 上次 rebalance 时间 |
| `rebalanceCooldown` | rebalance 冷却时间 |
| `maxSlippageBps` | add/remove/swap 的最大滑点 |
| `depositRatioToleranceBps` | 用户入金比例相对 `targetRatioBps` 的最大偏差 |
| `targetRatioBps` | 目标 token0/token1 价值比例，默认根据当前策略配置 |
| `minInitialDepositValue` | 首次入金的最小价值，避免 dust 级首次份额攻击 |
| `pausedDeposit` | 是否暂停 deposit |
| `pausedRebalance` | 是否暂停 rebalance |

#### 函数接口

| 函数 | 描述 |
|---|---|
| `deposit(amount0, amount1, minShares, deadline)` | 存入双资产并铸造 shares |
| `withdraw(shares, minAmount0, minAmount1, deadline)` | 销毁 shares 并取回双资产 |
| `previewDeposit(amount0, amount1)` | 预估可获得 shares |
| `previewWithdraw(shares)` | 预估可取回 token0/token1 |
| `totalValue()` | 按 TWAP 估算 Vault 总价值，单位统一为 `VALUE_SCALE = 1e18` 的 token1 quote value |
| `idle0()` / `idle1()` | 查询 Vault 当前 idle token 余额 |
| `canRebalance(targetVenue, params)` | 查询是否满足 rebalance 条件 |
| `rebalance(targetVenue, params)` | 执行再平衡 |
| `setAdapter(venue, adapter)` | Admin 配置 Adapter |
| `setRiskParams(...)` | Admin 配置风险参数 |
| `pauseDeposit()` / `unpauseDeposit()` | 有限暂停 deposit |
| `pauseRebalance()` / `unpauseRebalance()` | 有限暂停 rebalance |

#### preview 语义

- `previewDeposit` 和 `previewWithdraw` 是只读估算，不保证后续交易一定得到相同结果。
- `previewDeposit` 必须使用与实际 deposit 相同的 `totalValue()`、Oracle 和舍入规则。
- `previewWithdraw` 按 `shares / totalSupply` 计算当前底层 token0/token1 比例，不应依赖 spot price。
- 如果 Oracle 未 ready、stale 或流动性不足，`previewDeposit` 必须 revert，而不是返回误导性结果。
- 用户保护必须依赖实际交易中的 `minShares`、`minAmount0`、`minAmount1` 和 `deadline`。
- preview 不得修改 Oracle，不得调用 `oracle.update()`；更新 Oracle 必须由调用方或 Keeper 在交易前单独完成。

#### deposit 规则

```text
require(block.timestamp <= deadline)
require(!pausedDeposit)
require(amount0 > 0 && amount1 > 0)
require(token0/token1 are supported)
require(oracle.isReady() && !oracle.isStale())
require(oracle.hasSufficientLiquidity())
require(deposit value ratio within targetRatioBps +/- depositRatioToleranceBps)

balance0Before = token0.balanceOf(address(this))
balance1Before = token1.balanceOf(address(this))
totalValueBefore = totalValue()
transfer token0/token1 from user
actual0 = token0.balanceOf(address(this)) - balance0Before
actual1 = token1.balanceOf(address(this)) - balance1Before

require(actual0 == amount0 && actual1 == amount1)
valueIn = oracle.valueOf(actual0, actual1)
if totalSupply == 0:
  shares = valueIn * SHARE_SCALE / initialSharePrice
else:
  shares = valueIn * totalSupply / totalValueBefore
require(shares >= minShares)

mint shares to user
```

规则：

- `SHARE_SCALE = 1e18`，Vault shares 使用 18 decimals。
- 首次 deposit 使用 `initialSharePrice = 1e18`，其中 shares 和 quote value 都按 `VALUE_SCALE = 1e18` 归一化。
- 首次 shares 公式固定为 `shares = valueIn * SHARE_SCALE / initialSharePrice`。
- 非首次 shares 公式固定为 `shares = valueIn * totalSupply / totalValueBefore`，向下取整。
- 首次 deposit 的 `valueIn` 必须大于等于 `minInitialDepositValue`，避免极小初始存款操纵 share price。
- `targetRatioBps` 必须在部署时配置完成，首次 deposit 也必须满足入金比例约束。
- `minShares` 必须由用户传入，用于防止价格变化或舍入导致的损失。
- `targetRatioBps` 使用 token value 计算，不使用原始 token 数量计算，避免 decimals 不同导致比例错误。
- deposit 不自动 rebalance，避免用户 deposit 被复杂外部 AMM 调用影响。
- v1 不接受单边入金；如果用户只传入一种资产，必须 revert。
- deposit 使用 balance-delta 校验实际到账金额；如果 token 存在转账税、rebasing 或 fee-on-transfer 行为，必须 revert。
- deposit 必须 `nonReentrant`。
- `idle0/idle1` 以 token `balanceOf(vault)` 为准，不单独维护可被 desync 的 idle 存储变量。

#### withdraw 规则

```text
require(block.timestamp <= deadline)
require(shares > 0)
require(balanceOf(user) >= shares)

totalSupplyBefore = totalSupply
expected0, expected1 = previewWithdraw(shares)

if idle balance insufficient:
  remove proportional liquidity from current adapter

totalUnderlying0 = idle0AfterRemove + adapterAmount0AfterRemove
totalUnderlying1 = idle1AfterRemove + adapterAmount1AfterRemove
actual0 = mulDivDown(totalUnderlying0, shares, totalSupplyBefore)
actual1 = mulDivDown(totalUnderlying1, shares, totalSupplyBefore)
require(actual0 >= minAmount0)
require(actual1 >= minAmount1)

burn shares
transfer actual0/actual1 to user
```

规则：

- withdraw 必须按 shares 占比退出，不能让用户选择只提取高价值资产。
- withdraw 必须基于 `totalSupplyBefore` 计算份额比例，不能在 burn 后再用新的 `totalSupply` 计算。
- `actual0 = mulDivDown(totalUnderlying0, shares, totalSupplyBefore)`，`actual1 = mulDivDown(totalUnderlying1, shares, totalSupplyBefore)`。
- `actual0/actual1` 只能等于该份额比例对应的 idle + removed liquidity，不能使用或转出 Vault 全局余额。
- withdraw 输出按当前底层 token0/token1 数量比例计算，不使用 Oracle 价格决定两种资产的分配。
- 如果 Vault 当前资产在 AMM 中，必须按比例 remove liquidity。
- withdraw 应优先使用 idle balance，减少外部 AMM 调用。
- withdraw 不应被 Guardian 暂停，保证用户可以退出。
- withdraw 的 `minAmount0/minAmount1` 是用户保护参数，必须强制校验。
- withdraw 必须在 Adapter remove 完成后，使用最终实际可转出的 `actual0/actual1` 再次校验 `minAmount0/minAmount1`。
- withdraw 实际转出金额必须使用 balance-delta 和最终余额校验，防止 Adapter remove 后会计与实际余额不一致。
- 所有外部交互必须使用 `nonReentrant`。

---

### ILiquidityAdapter.sol

Adapter 将不同 AMM 的流动性操作统一成标准接口。

```solidity
interface ILiquidityAdapter {
    function venueId() external view returns (bytes32);

    function positionAmounts()
        external
        view
        returns (uint256 amount0, uint256 amount1);

    function addLiquidity(
        uint256 amount0,
        uint256 amount1,
        bytes calldata params
    ) external returns (uint256 used0, uint256 used1);

    function removeLiquidity(
        uint256 shareBps,
        bytes calldata params
    ) external returns (uint256 amount0, uint256 amount1);

    function collectFees(bytes calldata params)
        external
        returns (uint256 fee0, uint256 fee1);
}
```

Adapter 规则：

- Adapter 只能接收 Vault 调用。
- Adapter 不能保存用户 shares。
- Adapter 可以持有 AMM 仓位凭证，例如 V2 LP Token 或 V3 position NFT，但这些仓位的唯一受益人必须是 Vault。
- Adapter 不能把 token0/token1、LP Token、V3 NFT 或 fee 转给除 Vault 以外的地址。
- Adapter 必须对外部 AMM 调用使用 SafeERC20。
- Adapter 必须校验 pair/pool/token0/token1 与 Vault 配置一致。
- Adapter 操作完成后，未使用的 token0/token1 必须退回 Vault，不得长期滞留在 Adapter。
- Adapter 必须发出 add/remove/collect 事件，便于 fork 测试和链上审计。
- `positionAmounts()` 返回底层 token0/token1 数量，不返回 quote value；统一价值换算只能由 Vault 使用 Oracle 完成。
- `positionAmounts()` 返回当前仓位可估算的底层 token0/token1 数量。v1 不要求实现 V3 未领取手续费的只读估值。
- `removeLiquidity(shareBps, params)` 中 `shareBps` 的范围是 `1..10000`；取整产生的 dust 留在 Adapter 仓位中，后续 rebalance 或 withdraw 继续按比例处理。

---

### UniswapV2Adapter.sol

负责向指定 Uniswap V2 Pair 提供和移除流动性。

核心规则：

- pair 的 token0/token1 必须和 Vault token0/token1 一致。
- add liquidity 必须设置 `minAmount0`、`minAmount1`。
- remove liquidity 必须设置 `minAmount0`、`minAmount1`。
- Adapter 不做任意 swap，v1 只做 add/remove/collect。
- V2 LP Token 必须由 Adapter 或 Vault 持有，不能发送到外部地址。
- V2 Pair 必须来自配置的 factory，不能由调用方传入任意 pair。
- add/remove 后的未使用余额必须返回 Vault。
- Uniswap V2 手续费体现在 LP Token 对应的储备价值中，不存在独立 `collect`。
- V2Adapter 的 `collectFees()` 必须是 no-op，返回 `(0, 0)`，或仅发出 no-op 事件；真实收益只在 `positionAmounts()` 和 remove liquidity 时体现。

安全边界：

- 不支持 fee-on-transfer token。
- 不支持 rebasing token。
- 不支持 ERC777 callback token。
- 不依赖 Router 的 deadline 作为唯一保护，Adapter 自身也必须校验参数。

---

### UniswapV3Adapter.sol

负责管理指定 Uniswap V3 Pool 的集中流动性仓位。

核心状态：

| 状态 | 含义 |
|---|---|
| `pool` | V3 Pool 地址 |
| `positionManager` | NonfungiblePositionManager |
| `feeTier` | V3 fee tier，例如 500、3000、10000 |
| `tokenId` | 当前 V3 position NFT ID |
| `tickLower` / `tickUpper` | 当前仓位 tick range |
| `liquidity` | 当前 V3 position 的 liquidity |

核心规则：

- v1 至少支持一个 V3 fee tier。
- `tickLower/tickUpper` 必须满足 tick spacing。
- mint/increase/decrease/collect 必须设置 slippage bounds。
- V3 position NFT 必须由 Adapter 或 Vault 持有。
- collect fees 后资产回到 Vault 统一会计。
- V3 Adapter 必须实现 `IERC721Receiver`，且只接受来自 `NonfungiblePositionManager` 的本合约 position NFT。
- `tokenId == 0` 表示当前没有 V3 仓位；已有仓位时不得重复 mint 新 position，除非先 decrease/collect/close 旧 position。
- collect 费用时必须记录 fee0/fee1，并将实际到账金额转回 Vault。
- 为避免新用户在 V3 fees collect 前入金套利，Vault 在会改变 shares 的操作前必须先调用 `collectFees()`：
  - `deposit` 计算 `totalValueBefore` 前，如果当前是 V3 仓位，先 collect fees。
  - `withdraw` 计算可提取资产前，如果当前是 V3 仓位，先 collect fees。
  - `rebalance` 移除旧仓位前，先 collect fees。
- v1 不实现 V3 未领取手续费的只读估值；这是后续进阶版本的优化项。

tick range 计算：

```text
twapTick = oracle.getTwapTick()
width = strategy.tickRangeWidth
tickLower = floorToSpacing(twapTick - width)
tickUpper = floorToSpacing(twapTick + width)
```

边界：

- `tickLower < tickUpper`。
- `tickLower` 和 `tickUpper` 必须在 Uniswap V3 支持范围内。
- 高波动时应扩大 tick range 或拒绝 rebalance。
- stale oracle 时不得计算新 tick range。
- `getCurrentTick()` 只能用于波动率或偏离度判断，不能直接作为建仓中心 tick。

---

### TWAPOracle.sol

TWAP Oracle 读取 Uniswap V3 `observe()` 数据，计算指定窗口内的平均 tick、平均价格和波动率信号。

核心状态：

| 状态 | 含义 |
|---|---|
| `pool` | V3 Pool 地址 |
| `period` | TWAP 窗口，例如 30 minutes |
| `stalenessThreshold` | 允许的最大更新时间间隔 |
| `lastUpdated` | 最近一次更新 |
| `minLiquidity` | 最小 V3 in-range liquidity 门槛，单位为 Uniswap V3 `pool.liquidity()` 返回值 |
| `allowedPool` | Oracle 绑定的唯一 V3 Pool |
| `lastTwapTick` | 最近一次 `update()` 固化的 TWAP tick |
| `lastVolatilityBps` | 最近一次 `update()` 固化的波动率 bps |

函数：

| 函数 | 描述 |
|---|---|
| `update()` | 读取 V3 `observe()` 并固化 TWAP tick、波动率和 `lastUpdated` |
| `consult(amountIn, tokenIn)` | 返回 TWAP quote |
| `valueOf(amount0, amount1)` | 返回 token0/token1 组合的 `VALUE_SCALE` 精度 token1 quote value |
| `getTwapTick()` | 返回 TWAP tick |
| `getCurrentTick()` | 返回当前 tick |
| `getVolatility()` | 返回当前价格相对 TWAP 的偏离 bps |
| `isReady()` | 是否完成初始化 |
| `isStale()` | 是否过期 |
| `hasSufficientLiquidity()` | 是否满足最小流动性 |

TWAP 规则：

- `period` 必须大于 0。
- oracle 初始化后必须等待完整 period 才能作为有效价格源。
- `update()` 是 permissionless，任何人可以调用。
- `update()` 成功后必须设置 `lastUpdated = block.timestamp`，并发出 `OracleObservationUpdated(twapTick, volatilityBps, timestamp)`。
- `isReady == false` 时，`consult` 必须 revert。
- `isStale == true` 时，`consult`、`valueOf` 和 rebalance 必须 revert。
- `hasSufficientLiquidity == false` 时，`consult` 必须 revert。
- `consult` 只支持 `token0 -> token1` 或 `token1 -> token0`，其他 token 必须 revert。
- `amountIn == 0` 必须 revert。
- quote 使用 token 最小单位，不假设 token decimals 都是 18。
- `getVolatility()` 返回 bps，计算方式必须在测试中固定，不允许用调用方传入的价格。
- `valueOf(amount0, amount1)` 必须使用最近一次固化的 TWAP quote，不得使用 spot price。
- Admin 修改 `period/stalenessThreshold/minLiquidity` 时必须发出 `OracleConfigUpdated(...)`。

风险说明：

- TWAP 只能提高操纵成本，不能消除操纵风险。
- 低流动性、短窗口、刚初始化的池子仍然危险。
- v1 不允许将该 Oracle 用于借贷清算等高安全价格场景。

---

### VaultMath.sol

统一处理 shares、价值、比例和舍入。

要求：

- 所有除法必须明确舍入方向。
- 用户 mint shares 时向下取整。
- 用户 withdraw 输出时向下取整。
- Vault 价值计算不能凭空增加资产。
- dust 留在 Vault 中，不能被重复领取。

常用函数：

| 函数 | 描述 |
|---|---|
| `toShares(valueIn, totalValue, totalSupply)` | value -> shares |
| `toInitialShares(valueIn, initialSharePrice)` | 首次 deposit 的 value -> shares |
| `toValue(shares, totalValue, totalSupply)` | shares -> value |
| `mulDivDown(a,b,den)` | 向下取整乘除 |
| `mulDivUp(a,b,den)` | 向上取整乘除 |
| `bps(value, bpsValue)` | bps 计算 |

shares 公式：

```text
SHARE_SCALE = 1e18
VALUE_SCALE = 1e18

if totalSupply == 0:
  shares = mulDivDown(valueIn, SHARE_SCALE, initialSharePrice)
else:
  shares = mulDivDown(valueIn, totalSupply, totalValue)
```

要求：

- `initialSharePrice` 默认 `1e18`，表示 `1 share` 对应 `1 VALUE_SCALE` quote value。
- `totalValue == 0 && totalSupply > 0` 必须 revert，不能继续 mint shares。
- `shares == 0` 必须 revert。

---

## 资金流与会计模型

### deposit 资金流

```text
User
  -> approve token0/token1
  -> AdaptiveLPVault.deposit
  -> Vault receives token0/token1
  -> Vault mints shares
```

### rebalance 资金流

```text
AdaptiveLPVault
  -> currentAdapter.collectFees
  -> currentAdapter.removeLiquidity
  -> targetAdapter.addLiquidity
  -> update currentVenue/currentAdapter
```

### withdraw 资金流

```text
User
  -> AdaptiveLPVault.withdraw(shares)
  -> Vault removes proportional liquidity if needed
  -> Vault burns shares
  -> Vault transfers token0/token1 to user
```

### 会计不变量

| 编号 | 不变量 |
|---|---|
| A-1 | `totalSupply == 0` 时，所有用户 shares 为 0 |
| A-2 | 任意用户 `shares / totalSupply` 表示其对 Vault 资产的比例权益 |
| A-3 | deposit 不得降低已有用户按比例拥有的资产价值，除非由明确舍入 dust 导致 |
| A-4 | withdraw 不得让用户取出超过其 shares 占比的资产 |
| A-5 | Adapter 中资产必须属于 Vault，不属于调用 rebalance 的人 |
| A-6 | rebalance 前后总资产不应减少超过配置的 `maxSlippageBps` |
| A-7 | fee collection 后资产必须进入 Vault 会计 |
| A-8 | Vault shares 总量不因 rebalance 或 collect fees 改变 |
| A-9 | `totalValue = oracle.valueOf(idle0, idle1) + oracle.valueOf(adapterAmount0, adapterAmount1)` |

### 费用归属

- AMM 产生的手续费归 Vault 全体 share holder 按份额共享。
- `collectFees` 不铸造 shares，也不向 rebalancer 支付奖励。
- fees 收回后进入 Vault idle balance，并在下一次 `totalValue()` 中体现。
- V3 fees 必须在 deposit、withdraw、rebalance 前 collect 到 Vault，避免新用户在 collect 前入金分享历史费用。
- V2 没有独立 collect，手续费通过 LP Token 底层储备价值自然反映。
- v1 不收 performance fee 或 management fee；如未来增加，必须单独设计 fee receiver、费率上限、事件和测试。

---

## 再平衡策略

### 再平衡目标

Vault 根据 TWAP 和波动率信号，在不同 Venue 之间移动资产。

示例策略：

| 市场状态 | 目标 Venue |
|---|---|
| 低波动、价格稳定 | Uniswap V3 窄 tick range |
| 中等波动 | Uniswap V3 宽 tick range |
| 高波动 | Uniswap V2 或 IDLE |
| Oracle stale | 禁止 rebalance |
| 流动性不足 | 禁止 rebalance |

### `canRebalance`

```text
require(!pausedRebalance)
require(block.timestamp >= lastRebalanceAt + rebalanceCooldown)
require(oracle.isReady())
require(!oracle.isStale())
require(oracle.hasSufficientLiquidity())

volatilityBps = oracle.getVolatility()
expectedVenue = strategy.chooseVenue(volatilityBps)

require(targetVenue == expectedVenue)
require(targetVenue == IDLE || adapterOf[targetVenue] != address(0))
require(targetVenue != currentVenue)
```

目标 Venue 规则：

| 条件 | 目标 Venue |
|---|---|
| `volatilityBps < lowVolatilityBps` | V3 |
| `lowVolatilityBps <= volatilityBps < highVolatilityBps` | V2 |
| `volatilityBps >= highVolatilityBps` | IDLE |

规则：

- v1 使用固定阈值选择目标 Venue，不做复杂收益预测。
- `lowVolatilityBps`、`highVolatilityBps` 由 Admin 配置。
- 如果 Oracle stale 或流动性不足，禁止 rebalance。
- 测试必须覆盖每个 volatility bucket 和每个 Venue 转换。

返回值：

```solidity
function canRebalance(bytes32 targetVenue, bytes calldata params)
    external
    view
    returns (bool canRebalance, string memory reason);
```

### `rebalance`

```text
can, reason = canRebalance(targetVenue, params)
require(can)

valueBefore = totalValue()
collect fees
remove liquidity from current venue
add liquidity to target venue
valueAfter = totalValue()
require(valueAfter > 0)
update currentVenue/currentAdapter
update lastRebalanceAt
emit Rebalanced(...)
```

规则：

- rebalance 必须 `nonReentrant`。
- rebalance 不能指定任意接收地址。
- rebalance 不能把资产转给 rebalancer。
- rebalance 必须使用由合约根据 `maxSlippageBps` 和 TWAP quote 计算出的 slippage bounds，不能完全信任调用者传入的 `params`。
- `params` 只能包含已白名单 Adapter 所需的 tick range、deadline 或 min amounts，且必须经过 Vault 二次校验。
- `deadline` 由 Vault 使用 `block.timestamp + rebalanceDeadlineWindow` 生成，不能由 rebalancer 传入任意长 deadline。
- rebalance 失败时应整体 revert，不能留下半完成状态。
- 如果 remove 成功但 add 失败，同一交易必须全部回滚。
- rebalance 不铸造、不销毁 shares。
- `currentVenue/currentAdapter/lastRebalanceAt` 必须在外部 AMM 调用成功后更新。
- rebalance 前后必须记录 `valueBefore` 和 `valueAfter`，若 `valueAfter < valueBefore * (10000 - maxSlippageBps) / 10000` 必须 revert。
- `targetVenue == IDLE` 时不需要 Adapter；此时所有支持资产应回到 Vault idle balance。
- rebalance 后 Vault 和 Adapter 的 token0/token1 实际余额必须能被 `totalValue()` 覆盖，不能出现未计入会计的资产。

---

## 权限与角色

### 角色定义

| 角色 | 权限 |
|---|---|
| Owner / multisig | 配置 Adapter、Oracle、风险参数、暂停和恢复 |
| Depositor | deposit、withdraw |
| Rebalancer | permissionless 调用 `canRebalance` 和 `rebalance` |
| Guardian | 紧急暂停 deposit 或 rebalance |

### 权限规则

- Owner 在测试网可以由 deployer 持有；更正式的部署建议转移给 multisig。
- Rebalancer 是普通调用者，不应拥有角色权限或资产转移权限。
- Guardian 只能暂停，不能转移资产、修改 adapter、修改 oracle。
- Admin 参数修改必须发出事件。
- 替换 Adapter 前必须先把旧 Adapter 的仓位移回 IDLE。
- 替换 Oracle 前必须确认新 Oracle `isReady()` 且不是 stale。
- `withdraw` 不受 Guardian 暂停影响，保证用户可以退出。

### 参数表

| 参数 | 默认值 | 范围 | 修改权限 | 生效 |
|---|---:|---|---|---|
| `rebalanceCooldown` | 1 day | 1 hour / 7 days | Owner | 下一次 rebalance |
| `maxSlippageBps` | 100 bps | 0 / 500 bps | Owner | 下一次 add/remove |
| `depositRatioToleranceBps` | 50 bps | 0 / 1000 bps | Owner | 下一次 deposit |
| `minInitialDepositValue` | 部署配置 | 大于 0 | Owner | 下一次首次 deposit |
| `rebalanceDeadlineWindow` | 5 minutes | 1 minute / 30 minutes | Owner | 下一次 rebalance |
| `twapPeriod` | 30 minutes | 5 minutes / 24 hours | Owner | 下一次 update |
| `stalenessThreshold` | 2 * period | period / 7 days | Owner | 下一次 consult |
| `minLiquidity` | 部署配置 | 大于 0 | Owner | 下一次 consult |
| `tickRangeWidth` | 部署配置 | 大于 0，且满足 tick spacing | Owner | 下一次 V3 rebalance |
| `lowVolatilityBps` | 部署配置 | 小于 `highVolatilityBps` | Owner | 下一次 rebalance |
| `highVolatilityBps` | 部署配置 | 大于 `lowVolatilityBps` | Owner | 下一次 rebalance |

---

## 事件设计

关键状态变化必须发出事件。

| 事件 | 触发场景 |
|---|---|
| `Deposited(user, amount0, amount1, shares)` | 用户 deposit 成功 |
| `Withdrawn(user, shares, amount0, amount1)` | 用户 withdraw 成功 |
| `Rebalanced(executor, oldVenue, newVenue, valueBefore, valueAfter, timestamp)` | rebalance 成功 |
| `FeesCollected(adapter, amount0, amount1)` | Adapter collect fees 成功 |
| `AdapterUpdated(venue, oldAdapter, newAdapter)` | Owner 更新 Adapter |
| `RiskParamsUpdated(param, oldValue, newValue)` | Owner 更新风险参数 |
| `DepositPaused(account)` / `DepositUnpaused(account)` | deposit 暂停或恢复 |
| `RebalancePaused(account)` / `RebalanceUnpaused(account)` | rebalance 暂停或恢复 |
| `OracleObservationUpdated(twapTick, volatilityBps, timestamp)` | Oracle 更新观测值 |
| `OracleConfigUpdated(param, oldValue, newValue)` | Oracle 参数更新 |

事件要求：

- 事件必须包含足够定位问题的关键参数。
- 参数变更事件必须包含 old value 和 new value。
- 不得只依赖链下日志记录关键状态变化。

---

## 错误与 Revert 策略

工程化实现应优先使用自定义错误，降低 gas 并让测试能精确断言失败原因。

| 错误 | 场景 |
|---|---|
| `DeadlineExpired()` | 用户或 Vault 生成的 deadline 已过期 |
| `Paused(bytes32 action)` | deposit 或 rebalance 被暂停 |
| `InvalidAmount()` | amount 为 0、shares 为 0 或计算结果为 0 |
| `UnsupportedToken(address token)` | token 不是 Vault 配置资产 |
| `InvalidDepositRatio()` | 双资产入金价值比例超过容忍区间 |
| `OracleNotReady()` | Oracle 初始化窗口未完成 |
| `OracleStale()` | Oracle 观测过期 |
| `InsufficientOracleLiquidity()` | V3 pool 流动性低于门槛 |
| `SlippageExceeded()` | add/remove/rebalance 后资产少于允许下限 |
| `UnauthorizedAdapter()` | Adapter 未被 Owner 配置 |
| `InvalidRecipient()` | AMM、Adapter 或 params 试图把资产发给非 Vault 地址 |
| `ValueLossExceeded()` | rebalance 后总价值损失超过 `maxSlippageBps` |

要求：

- 失败原因必须在单元测试中逐项覆盖。
- 禁止使用宽泛的 `require(false, "ERR")` 代替可定位的错误。
- 对外部 AMM 调用失败可以透传底层 revert，但 Vault 状态必须整体回滚。

---

## 安全模型

### 主要攻击面

| 风险 | 描述 | 缓解 |
|---|---|---|
| Oracle manipulation | 攻击者操纵价格触发错误 rebalance | TWAP、min liquidity、cooldown、max slippage |
| Reentrancy | AMM 或 token callback 重入 Vault | `nonReentrant`、不支持 ERC777、固定 recipient、整体回滚 |
| Slippage loss | rebalance 价格不利导致资产损失 | `minAmount0/minAmount1`、`maxSlippageBps` |
| Sandwich attack | 大额 rebalance 被夹 | slippage、TWAP、cooldown、限制目标 venue |
| Griefing | 反复触发无意义 rebalance | cooldown、目标 Venue 必须变化 |
| Rounding exploit | 小额反复 deposit/withdraw 偷 dust | 明确舍入、最小金额、invariant tests |
| Adapter misuse | Adapter 将资产转给错误地址 | adapter allowlist、recipient 固定为 Vault |
| Admin abuse | 管理员恶意改参数或替换错误 Adapter / Oracle | 多签、事件、参数范围、部署前检查 |

### Token 支持范围

v1 只支持标准 ERC-20。

规则：

- `token0/token1` 在 Vault 构造函数中设置为 immutable，不允许运行时替换。
- 部署前必须在 fork 测试中验证 token decimals、transfer、approve、balanceOf 行为。
- Vault 和 Adapter 不提供任意 token allowlist，v1 是固定资产对 Vault。

不支持：

- fee-on-transfer token。
- rebasing token。
- ERC777 callback token。
- 转账税 token。
- decimals 不稳定或非标准 `balanceOf` 行为的 token。

非 18 decimals token 可以支持，但必须在测试中覆盖。

### 暂停策略

| 功能 | 是否可暂停 | 说明 |
|---|---|---|
| deposit | 可暂停 | 防止异常资产继续流入 |
| rebalance | 可暂停 | 防止错误策略继续执行 |
| withdraw | 不建议暂停 | 用户应能取回资产 |
| admin config | 不由 Guardian 暂停 | 由 Owner 管理 |

---

## 测试计划

### 单元测试

| 模块 | 测试点 |
|---|---|
| Vault deposit | 首次 deposit shares 公式、非首次 deposit shares 公式、minShares、deadline、zero amount |
| Vault deposit ratio | 双资产比例偏差、单边入金 revert、fee-on-transfer mock revert |
| Vault withdraw | proportional withdraw、`totalSupplyBefore` 份额比例、minAmount、idle insufficient、burn shares、withdraw 不被 pause 影响 |
| Vault accounting | totalValue、share price、dust、rounding |
| TWAPOracle | ready、stale、period、volatility、min liquidity |
| V2Adapter | add/remove liquidity、wrong pair、wrong factory、slippage、unused balance returned、collectFees no-op |
| V3Adapter | tick range、mint/increase/decrease/collect、wrong pool、wrong NFT sender |
| Rebalance | canRebalance false/true、cooldown、stale oracle、slippage、params 校验、deadline |
| Access control | onlyOwner、Guardian pause、rebalancer restrictions |
| Events | deposit、withdraw、rebalance、adapter update、risk param update 事件参数正确 |
| Errors | 自定义错误 selector 与失败场景一一对应 |

### 集成测试

- 用户 deposit WETH/USDC。
- Vault 保持 IDLE。
- Rebalance 到 Uniswap V2。
- 检查 V2 收益通过 LP Token 底层价值体现。
- Rebalance 到 Uniswap V3。
- 收集 V3 fees，并检查费用进入 Vault idle balance。
- 用户 withdraw。
- 检查用户拿回资产与 shares 占比一致。

### Fork 测试

主网 fork 上建议使用真实地址：

| 资产 / 合约 | 用途 |
|---|---|
| WETH | token0 |
| USDC | token1 |
| Uniswap V2 Router / Factory | V2 流动性 |
| Uniswap V3 Pool WETH/USDC | TWAP / V3 流动性 |
| NonfungiblePositionManager | V3 position 管理 |

要求：

- fork block 固定，保证测试可复现。
- 测试真实 V3 oracle observe。
- 测试真实 V2 add/remove。
- 不依赖实时主网价格波动。

### 不变量测试

- 总 shares 不凭空增加。
- 用户 withdraw 不能超过其 shares 占比。
- Rebalance 不改变 shares 总量。
- Adapter 不能把资产转给非 Vault 地址。
- `totalValue` 在允许滑点范围内保持守恒。
- V3 fees 在 deposit/withdraw/rebalance 前被 collect，不能被新用户套利。
- permissionless rebalance 不能通过恶意 params 改变目标 Adapter 或收款地址。
- deposit/withdraw 反复执行不能通过舍入持续抽取 Vault dust。

---

## 部署指南

### 前置要求

| 工具 | 版本 |
|---|---|
| Solidity | `foundry.toml` 固定 solc patch 版本，例如 `0.8.20` |
| Foundry | 记录完整 `forge --version` 输出 |
| OpenZeppelin | 固定 commit 或明确版本 |
| Uniswap V2/V3 interfaces | 固定依赖 commit 或明确版本 |

依赖规则：

- 不允许主网部署使用浮动依赖。
- `foundry.toml` 必须固定 optimizer、optimizer runs、evm version。
- fork 测试必须固定 block number。
- 每次修改 Adapter 或 Oracle 后必须重新运行 fork 测试。

### 环境变量

```bash
SEPOLIA_RPC_URL=
MAINNET_RPC_URL=
DEPLOYER_PRIVATE_KEY=
ETHERSCAN_API_KEY=
```

### 部署顺序

1. 确认 token0/token1，并完成 token 行为测试。
2. 部署 TWAPOracle，并配置 `period/stalenessThreshold/minLiquidity`。
3. 等待完整 TWAP period 后调用 `oracle.update()`，确认 `isReady && !isStale && hasSufficientLiquidity`。
4. 部署 UniswapV2Adapter。
5. 部署 UniswapV3Adapter。
6. 部署 AdaptiveLPVault，写入 token、oracle、初始风险参数。
7. 配置 adapter allowlist。
8. 配置 Guardian。
9. 可选：转移 owner 到 multisig。
10. 运行 fork dry-run。
11. Sepolia 部署并验证合约。

### 部署前检查

| 检查项 | 要求 |
|---|---|
| token 支持 | token0/token1 为标准 ERC-20，非 fee-on-transfer/rebasing/ERC777 |
| Oracle | TWAP period、stalenessThreshold、minLiquidity 配置完成 |
| Adapter | V2 pair、V3 pool、position manager 地址正确 |
| 权限 | Owner / Guardian 配置正确 |
| Pause | Guardian 只能暂停 deposit/rebalance，不能暂停 withdraw |
| 测试 | unit、integration、fork、invariant 全部通过 |
| 文档 | README、SECURITY.md、NatSpec 完整 |

---

## 升级与迁移策略

v1 默认不使用代理合约。核心 Vault 按不可升级合约部署，通过部署新 Vault、新 Adapter 或新 Oracle 完成迁移。

### 可迁移模块

| 模块 | 策略 |
|---|---|
| Vault | 不原地升级。部署新 Vault，用户自行 withdraw 后 deposit 到新 Vault |
| V2 Adapter | 可由 Owner 替换白名单 Adapter，但替换前必须从旧 Adapter 移除全部仓位 |
| V3 Adapter | 可由 Owner 替换白名单 Adapter，但旧 `tokenId` 必须 decrease/collect/close |
| TWAPOracle | 可由 Owner 替换，但替换前必须完成 ready/stale/minLiquidity 测试 |
| Risk Params | 由 Owner 修改，必须发出事件 |

### 迁移流程

1. Guardian 暂停 deposit 和 rebalance。
2. 保持 withdraw 开放。
3. collect fees。
4. remove liquidity 到 IDLE。
5. 部署新模块。
6. fork dry-run 验证新配置。
7. Owner 更新 Adapter / Oracle 或引导用户迁移到新 Vault。
8. 发出迁移公告并保留旧 Vault withdraw。

### 禁止事项

- 不允许 Admin 直接转移用户资产。
- 不允许强制把用户 shares 迁移到新 Vault。
- 不允许在旧 Adapter 仍有仓位时切换 `currentAdapter`。
- 不允许暂停 withdraw 作为常规迁移手段。

---

## 数值精度与舍入

| 场景 | 舍入方向 |
|---|---|
| mint shares | 向下取整，避免给新用户多发 shares |
| withdraw amounts | 向下取整，避免取出超过权益的资产 |
| slippage min amount | 向下取整，保护交易可执行性但必须满足用户下限 |
| TWAP quote | 按 Oracle 实现固定舍入，并在测试中断言 |
| fee accounting | 以实际到账 balance delta 为准 |

要求：

- `VALUE_SCALE = 1e18`，所有 quote value 都归一化到 18 decimals 后再参与 shares 计算。
- `token0/token1` 原始金额仍按各自最小单位存储和转账，不修改 token decimals。
- `oracle.valueOf(amount0, amount1)` 返回 `VALUE_SCALE` 精度的 token1 quote value。
- 所有 token 金额按最小单位处理。
- 不假设 token decimals 都是 18。
- 所有 `mulDiv` 必须明确使用 up 或 down。
- 小额导致 shares 为 0 或 amountOut 为 0 时必须 revert。

---

## 验收标准

### 功能验收

| 类别 | 标准 |
|---|---|
| Deposit / Withdraw | 正常路径和失败路径全部通过 |
| Share accounting | 首次、非首次、多用户、dust 测试通过 |
| V2 Adapter | add/remove liquidity 通过 |
| V3 Adapter | 至少一个 fee tier 的 position 管理通过 |
| V3 fee accounting | collect fees 后 balance-delta 对齐 |
| TWAP | ready/stale/min liquidity/volatility 测试通过 |
| Rebalance | IDLE -> V2、V2 -> V3、V3 -> IDLE 至少覆盖两条 |
| Security | reentrancy、slippage、access control、rounding 测试通过 |
| Fork test | 固定 block 的主网 fork 测试通过 |
| Events | 关键状态变化事件完整 |

### 覆盖率验收

- 总行覆盖率不低于 80%。
- Vault、Adapter、Oracle 核心路径覆盖率不低于 90%。
- 所有权限和失败路径必须有负向测试。

### Gas 验收

- `forge test --gas-report` 必须生成报告。
- `deposit`、`withdraw`、`rebalance`、`V3 mint`、`V3 collect` 必须记录 gas。
- 核心操作超过基准 20% 时，PR 或提交说明必须解释原因。

测量环境：

| 项目 | 要求 |
|---|---|
| Solidity | 与 `foundry.toml` 固定版本一致 |
| Optimizer | enabled，runs 固定，例如 200 |
| EVM version | 固定，例如 paris 或 shanghai |
| Foundry | 使用 README 记录的完整 `forge --version` |
| Fork block | 固定 block number |
| 初始状态 | 至少包含 IDLE、V2 active、V3 active 三类场景 |

### 文档验收

- README 能说明项目目标、架构、使用方式和安全边界。
- 所有 public / external 函数有 NatSpec。
- SECURITY.md 至少列出 3 个风险和缓解措施。
- 部署脚本有明确参数说明。

---

## 交付清单

### 核心合约

| 文件 | 交付内容 |
|---|---|
| `src/AdaptiveLPVault.sol` | 双资产 Vault、ERC-20 shares、deposit、withdraw、rebalance 调度 |
| `src/interfaces/ILiquidityAdapter.sol` | AMM Adapter 标准接口 |
| `src/adapters/UniswapV2Adapter.sol` | Uniswap V2 add/remove liquidity，V2 fees 通过 LP 底层价值体现 |
| `src/adapters/UniswapV3Adapter.sol` | Uniswap V3 position mint/increase/decrease/collect 和 fee accounting |
| `src/TWAPOracle.sol` | V3 TWAP、stale 检查、min liquidity 检查 |
| `src/libraries/VaultMath.sol` | shares、value、bps、rounding、mulDiv 计算 |
| `src/libraries/TickMathHelper.sol` | V3 tick range、tick spacing、边界检查 |

### 测试交付

| 目录 | 交付内容 |
|---|---|
| `test/unit/` | Vault、Oracle、V2 Adapter、V3 Adapter、权限、错误路径单元测试 |
| `test/integration/` | IDLE、V2、V3、rebalance、withdraw 端到端流程测试 |
| `test/fork/` | 固定 block 的主网 fork 测试，覆盖真实 V2/V3 交互 |
| `test/invariant/` | shares、totalValue、rounding、permissionless rebalance 不变量测试 |

### 脚本与文档

| 文件 | 交付内容 |
|---|---|
| `script/Deploy.s.sol` | 部署 Vault、Oracle、Adapter、权限和风险参数 |
| `script/DeployTestnet.s.sol` | 测试网部署和合约验证脚本 |
| `foundry.toml` | 固定 Solidity、optimizer、EVM version 和依赖配置 |
| `README.md` | 协议设计、模块 PRD、资金流、安全边界和验收标准 |
| `SECURITY.md` | 风险说明、漏洞响应、已知限制和安全假设 |

### 最小可验收版本

- `AdaptiveLPVault` 支持双资产 deposit / withdraw，shares 会计测试通过。
- `UniswapV2Adapter` 支持 add/remove liquidity，fork 测试通过。
- `TWAPOracle` 支持 ready/stale/min liquidity 检查。
- `rebalance` 至少覆盖 `IDLE -> V2` 和 `V2 -> IDLE`。
- 所有 public / external 函数补齐 NatSpec。
- `forge test`、`forge coverage`、`forge test --gas-report` 通过。

### 完整版本

- 支持 `UniswapV3Adapter` 的一个 fee tier。
- 支持 V3 tick range 计算。
- 支持 V3 collect fees，并通过 collect 前后 balance-delta 测试。
- 覆盖 `IDLE -> V2`、`V2 -> V3`、`V3 -> IDLE` 再平衡路径。
- 完成 Sepolia 部署、Etherscan 验证和部署参数记录。

---

## 项目亮点

- 双资产 Vault shares 会计。
- Uniswap V2 / V3 Adapter Pattern。
- TWAP Oracle 与波动率信号。
- 受保护的再平衡流程。
- Slippage、Reentrancy、Access Control、Rounding 等安全边界。
- Foundry 单元测试、集成测试、主网 fork 测试和不变量测试。

---

## 许可证

本项目采用 MIT License。
