# Adaptive LP Vault

> 一个自再平衡流动性 Vault 项目。用户存入 WETH / USDC 两种 ERC-20 资产，Vault 铸造 ERC-20 份额，并根据 TWAP 和波动率信号，在 IDLE、Uniswap V2 和一个 Uniswap V3 fee tier 之间切换流动性仓位。
>
> 本项目聚焦 Vault 份额会计、AMM Adapter 架构、TWAP 风控、自动再平衡、安全边界和 Foundry 测试体系。

## 项目范围

本项目面向测试网部署与主网 fork 验证，目标是形成一套围绕双资产 Vault、AMM adapter、TWAP 风控和再平衡流程的可验证合约实现；当前版本未经审计，不建议直接用于真实主网资金。

核心范围包括：

- ERC-20 Vault shares 会计。
- 双资产 deposit / withdraw 流程。
- Venue Adapter Pattern。
- Uniswap V2 流动性添加和移除。
- 一个 Uniswap V3 fee tier 的 position 管理。
- Uniswap V3 TWAP Oracle 读取和波动率计算。
- 基于波动率的 rebalance 决策。
- Slippage protection、reentrancy protection、access control、emergency controls。
- Foundry 单元测试和 Fork 测试。
- Sepolia 部署与合约验证流程。

## 非目标

以下内容不属于当前核心版本范围：

- 多个 V3 position。
- 多个 V3 fee tier。
- Curve、Balancer 或跨 DEX 优化。
- Rebalancer 激励机制。
- DAO / Governance 参数控制。
- 可升级代理合约架构。
- 生产级 Vault 迁移方案。
- 必做的 invariant testing。
- 必做的 gas benchmark 优化报告。
- 支持 fee-on-transfer、rebasing、ERC777 或非标准 ERC-20 token。

这些内容可以作为后续扩展，但不应该阻塞当前核心版本的交付。

## 合约架构

```text
Users
  -> AdaptiveLPVault
      -> VaultMath
      -> TWAPOracle
      -> IVenueAdapter
          -> UniswapV2Adapter
          -> UniswapV3Adapter
      -> WETH / USDC

Owner / Admin
  -> 配置 adapters
  -> 配置 oracle 和 risk parameters
  -> 暂停 / 恢复 deposit 和 rebalance
  -> emergencyExit 回到 IDLE

Rebalancer / Anyone
  -> canRebalance()
  -> rebalance()
```

## 合约结构

```text
src/
├── AdaptiveLPVault.sol
├── TWAPOracle.sol
├── interfaces/
│   └── IVenueAdapter.sol
├── adapters/
│   ├── UniswapV2Adapter.sol
│   └── UniswapV3Adapter.sol
├── libraries/
│   └── VaultMath.sol
└── mocks/
    ├── MockERC20.sol
    ├── MockV2Pair.sol
    └── MockV3Pool.sol

test/
├── Vault.t.sol
├── Oracle.t.sol
├── V2Adapter.t.sol
├── V3Adapter.t.sol
├── Rebalance.t.sol
└── Fork.t.sol

script/
└── Deploy.s.sol

README.md
SECURITY.md
```

## Vault 模型

`AdaptiveLPVault` 接收两种资产，例如 WETH 和 USDC，并铸造 ERC-20 Vault shares。

Vault shares 的用户体验参考 ERC-4626，但本项目不是严格 ERC-4626 实现，因为 ERC-4626 是单资产 Vault 标准，而本项目是双资产 Vault。

### 支持的 Venue

```solidity
enum Venue {
    IDLE,
    UNISWAP_V2,
    UNISWAP_V3
}
```

当前版本只需要支持一个 Uniswap V3 fee tier，例如 WETH / USDC 0.30%。

## 核心用户流程

### Depositor

- 用户 approve WETH 和 USDC。
- 调用 `deposit(amount0, amount1, minShares, deadline)`。
- 获得 Vault shares。
- 后续调用 `withdraw(shares, minAmount0, minAmount1, deadline)`。
- 按 shares 占比取回 WETH 和 USDC。

### Rebalancer

- 调用 `canRebalance(targetVenue)`。
- 如果允许，则调用 `rebalance(targetVenue, params)`。
- Vault 退出旧 venue，并进入新 venue。
- Rebalancer 不能指定任意 token、adapter 或 recipient。

### Admin

- 配置 adapter 地址。
- 配置风险参数。
- 紧急情况下暂停 deposit 和 rebalance。
- 调用 `emergencyExit()` 将流动性撤回 IDLE。
- 不能直接转移用户资产。

## AdaptiveLPVault.sol

### 职责

- 持有用户资产。
- 铸造和销毁 Vault shares。
- 记录当前 liquidity venue。
- 通过 adapters 调度 add/remove liquidity。
- 从 oracle 读取 TWAP 和 volatility。
- 校验 rebalance 条件。
- 校验 slippage、deadline、pause 和权限规则。

### 主要状态

| 状态 | 说明 |
|---|---|
| `token0` | 第一个 Vault 资产，例如 WETH |
| `token1` | 第二个 Vault 资产，例如 USDC |
| `oracle` | TWAP Oracle 合约 |
| `currentVenue` | 当前流动性场所 |
| `adapters[venue]` | 每个 venue 对应的白名单 adapter |
| `lastRebalance` | 上次成功 rebalance 时间 |
| `cooldown` | 两次 rebalance 之间的最小间隔 |
| `maxSlippageBps` | rebalance 允许的最大价值损失 |
| `paused` | deposit 和 rebalance 是否暂停 |

### 必要函数

| 函数 | 说明 |
|---|---|
| `deposit(amount0, amount1, minShares, deadline)` | 存入双资产并铸造 shares |
| `withdraw(shares, minAmount0, minAmount1, deadline)` | 销毁 shares 并按比例提款 |
| `previewDeposit(amount0, amount1)` | 预估可获得 shares |
| `previewWithdraw(shares)` | 预估可取回 token 数量 |
| `totalAssets()` | 返回 Vault + Adapter 中的 token0/token1 总资产 |
| `totalValue()` | 返回 TWAP 计价后的总价值 |
| `canRebalance(targetVenue)` | 判断当前是否允许 rebalance |
| `rebalance(targetVenue, params)` | 从当前 venue 切换到目标 venue |
| `setAdapter(venue, adapter)` | Owner 配置 adapter |
| `setRiskParams(...)` | Owner 更新 cooldown/slippage 等参数 |
| `pause()` / `unpause()` | Owner 暂停或恢复 deposit/rebalance |
| `emergencyExit()` | Owner 将仓位撤回 IDLE 并暂停 |

### Deposit 规则

- `deadline` 不能过期。
- `amount0` 和 `amount1` 都必须大于 0。
- paused 时不能 deposit。
- Oracle 必须 ready 且不能 stale。
- 实际到账数量必须等于用户传入数量，不支持 fee-on-transfer token。
- Shares 根据 TWAP 归一化价值计算。
- `shares >= minShares`。
- 函数必须使用 `nonReentrant`。

### Withdraw 规则

- `shares` 必须大于 0。
- 用户必须持有足够 shares。
- 即使 Vault paused，也必须允许 withdraw。
- 如果 idle balance 不足，Vault 应从当前 adapter 按比例移除流动性。
- 用户按 shares 占比获得 token0/token1。
- `amount0 >= minAmount0` 且 `amount1 >= minAmount1`。
- 函数必须使用 `nonReentrant`。

## IVenueAdapter.sol

所有 venue 使用同一个 adapter 接口，让 Vault 不需要关心不同 AMM 的内部差异。

```solidity
interface IVenueAdapter {
    function venueId() external view returns (bytes32);

    function addLiquidity(
        uint256 amount0,
        uint256 amount1,
        bytes calldata params
    ) external returns (uint256 used0, uint256 used1);

    function removeLiquidity(
        uint256 liquidityBps,
        bytes calldata params
    ) external returns (uint256 amount0, uint256 amount1);

    function collectFees(bytes calldata params)
        external
        returns (uint256 fee0, uint256 fee1);

    function positionValue()
        external
        view
        returns (uint256 amount0, uint256 amount1);

    function hasPosition() external view returns (bool);
}
```

Adapter 要求：

- 只有 Vault 可以调用会改变状态和移动资产的 adapter 函数。
- Adapter 不能把用户资产转给任意 recipient。
- 未使用的 token 必须返回 Vault。
- Adapter 必须校验 token 和 pool/pair 地址。
- Add/remove/collect 操作必须 emit event。

## UniswapV2Adapter.sol

必要行为：

- 向配置好的 WETH / USDC V2 pair 或 router 添加流动性。
- 从配置好的 V2 venue 移除流动性。
- 将未使用 token 返回 Vault。
- 返回当前仓位的 token0/token1 估算数量。
- V2 手续费通过 LP token 底层价值增长体现。

安全规则：

- Pair/factory 必须由 owner 配置，不能由 rebalancer 任意传入。
- Recipient 必须始终是 Vault。
- Add/remove liquidity 必须使用 min amounts 或 Vault 层的 slippage/value-loss 检查。

## UniswapV3Adapter.sol

必要行为：

- 支持一个选定的 V3 fee tier。
- Mint 或 increase 一个 V3 position。
- Decrease liquidity 并 collect fees。
- 存储当前 `tokenId`。
- 如果 adapter 持有 V3 position NFT，需要实现 `IERC721Receiver`。
- Collected fees 和未使用 token 必须返回 Vault。

当前版本不需要支持多个 position、多个 fee tier 或复杂策略优化。

## TWAPOracle.sol

Oracle 读取 Uniswap V3 TWAP 数据，并提供 Vault rebalance 所需的 volatility 信号。

### 必要函数

| 函数 | 说明 |
|---|---|
| `update()` | 读取并保存最新 TWAP 和 volatility |
| `getTwapTick()` | 返回最新 TWAP tick |
| `getCurrentTick()` | 返回当前 pool tick |
| `getVolatility()` | 返回当前价格相对 TWAP 的偏离 bps |
| `getRecommendedVenue()` | 根据 volatility 返回推荐 venue |
| `isReady()` | 返回 oracle 是否已有足够 observation |
| `isStale()` | 返回 oracle 数据是否过期 |

### Volatility 到 Venue 的映射

| Volatility | 信号 | 推荐 Venue |
|---:|---|---|
| `< 50 bps` | 非常稳定 | V3 |
| `50-150 bps` | 正常 | V3 |
| `150-300 bps` | 波动升高 | V2 |
| `> 300 bps` | 高波动 | IDLE |

阈值可以由 owner 配置，但测试必须覆盖每个 bucket 是否返回正确 venue。

## Rebalance 逻辑

### canRebalance

`canRebalance(targetVenue)` 应检查：

- Vault 没有 paused。
- Cooldown 已经过。
- Target venue 不等于 current venue。
- Oracle ready 且不 stale。
- Target venue 等于 oracle 推荐 venue。
- 如果实现 gas protection，则 gas price 低于配置上限。
- 如果 target 不是 IDLE，则目标 adapter 已配置。

### rebalance

`rebalance(targetVenue, params)` 应执行：

1. 调用 `canRebalance(targetVenue)`。
2. 记录 rebalance 前的总价值。
3. 如有需要，从当前 adapter collect fees。
4. 从当前 venue 移除流动性。
5. 如果 target 不是 IDLE，则向目标 venue 添加流动性。
6. 记录 rebalance 后的总价值。
7. 如果价值损失超过 `maxSlippageBps`，则 revert。
8. 更新 `currentVenue` 和 `lastRebalance`。
9. Emit `Rebalanced`。

Rebalance 必须是原子操作。如果进入新 venue 失败，整笔交易必须回滚。

## 安全要求

### Reentrancy Protection

所有会移动资产的 external state-changing 函数都应使用 `nonReentrant`：

- `deposit`
- `withdraw`
- `rebalance`
- `emergencyExit`

### Slippage Protection

Rebalance 不能接受任意 slippage。

Vault 应比较 rebalance 前后的价值：

```text
valueAfter >= valueBefore * (10_000 - maxSlippageBps) / 10_000
```

### Access Control

- Owner 可以配置 adapter、oracle 和风险参数。
- Rebalancer 只能触发已经通过校验的 rebalance。
- Rebalancer 不能修改 recipient、adapter、token 或策略参数。
- 用户始终可以 withdraw 自己按比例拥有的资产。

### Emergency Controls

- `pause()` 禁用 deposit 和 rebalance。
- `withdraw()` 在 paused 状态下仍然可用。
- `emergencyExit()` 从当前 adapter 移除流动性，将资产回到 IDLE，并 pause Vault。

## 需要考虑的攻击面

| 风险 | 缓解 |
|---|---|
| Oracle manipulation | 使用 TWAP 而不是 spot price，加入 stale check 和 cooldown |
| Reentrancy | `nonReentrant`、checks-effects-interactions、不支持 callback token |
| Griefing rebalance | Cooldown、recommended venue 校验、target 必须变化 |
| Sandwich attack | Slippage/value-loss 检查 |
| Rounding dust | 最小 shares、明确舍入规则、小额测试 |
| Admin mistake | Access control、events、参数文档 |

## 测试计划

测试覆盖率目标：至少 80%。

### 单元测试

| 测试文件 | 覆盖内容 |
|---|---|
| `Vault.t.sol` | deposit、withdraw、share accounting、deadline、min amounts |
| `Oracle.t.sol` | TWAP 读取、stale 状态、volatility buckets |
| `V2Adapter.t.sol` | add/remove liquidity、wrong pair、unused token return |
| `V3Adapter.t.sol` | mint/decrease/collect、tick range、NFT handling |
| `Rebalance.t.sol` | IDLE -> V2、V2 -> IDLE、V2 -> V3 或 V3 -> IDLE |

### Fork 测试

`Fork.t.sol` 应使用固定 fork block，验证合约与真实 Uniswap 合约的交互路径。

必要 fork 检查：

- Oracle 可以读取真实 V3 WETH / USDC pool。
- V2 adapter 可以 add/remove liquidity，或在真实 pair/router 上完成 dry-run。
- V3 adapter 可以 mint/decrease/collect，或完成受控 fork 交互。

### 负向测试

必须覆盖的失败路径：

- Deposit amount 为 0。
- Deadline 过期。
- `minShares` 不足。
- `minAmount0` / `minAmount1` 不足。
- 未授权 admin 操作。
- Oracle stale。
- Cooldown 内 rebalance。
- Target venue 不是 oracle 推荐 venue。
- Slippage 过大。
- 如果实现恶意 mock token，则覆盖 reentrancy attempt。

## 部署

部署部分应包含 Sepolia 部署和验证说明。

### 环境变量

```bash
SEPOLIA_RPC_URL=
PRIVATE_KEY=
ETHERSCAN_API_KEY=
```

### 部署步骤

1. 部署或选择 WETH / USDC 测试 token。
2. 部署 `TWAPOracle`。
3. 部署 `UniswapV2Adapter`。
4. 为一个 fee tier 部署 `UniswapV3Adapter`。
5. 部署 `AdaptiveLPVault`。
6. 在 Vault 中配置 adapters。
7. 配置风险参数。
8. 在 Etherscan 验证合约。
9. 在 README 或部署记录中写下 deployed addresses。

## 必交付内容

| 交付物 | 要求 |
|---|---|
| Smart contracts | Vault + Oracle + V2 adapter + 至少一个 V3 adapter |
| Test suite | Unit tests + fork tests，目标 80%+ coverage |
| Documentation | README 包含 setup、architecture、usage |
| Deployment | Sepolia 上验证过的合约 |
| Security analysis | SECURITY.md 至少写 3 个风险和缓解措施 |

## 评分项对齐

| 类别 | 权重 | 验收重点 |
|---|---:|---|
| Core Functionality | 35% | Deposit、withdraw、rebalance 正确工作 |
| TWAP Integration | 25% | Oracle 读取可用，volatility 计算合理 |
| Security | 20% | Reentrancy guard、slippage protection、access control |
| Testing | 15% | Fork tests 通过，edge cases 覆盖 |
| Documentation | 5% | README 清晰，NatSpec 注释完整 |

## 常见错误

- Rebalance 没有 slippage protection。
- 移动资产的函数缺少 reentrancy guard。
- 地址硬编码，导致无法部署到 Sepolia。
- 没有 fork tests。
- 忽略 dust 和 rounding。
- 在 V2/V3 基础流程没完成前过度扩展太多 venue。

## 后续扩展

以下是核心版本完成后的可选加分项：

- 多 V3 position 策略。
- 多 V3 fee tier。
- V3 liquidity adapter 和 NFT position 管理。
- Rebalancer incentives。
- Governance-controlled parameters。
- Invariant tests。
- Gas optimization report。
- 更严谨的 V3 fee accounting，防止新用户分享旧用户未领取 fees。
- Cross-DEX optimization。

## License

MIT
