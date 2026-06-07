# 安全分析

本文档说明 Adaptive LP Vault 项目的主要风险、安全假设和缓解措施。

本项目面向作品集展示、测试网部署和主网 fork 验证，尚未经过审计，不能用于真实主网资金。

## 支持的资产范围

Vault 只支持部署时配置的标准 ERC-20 token。

不支持以下 token：

- Fee-on-transfer token。
- Rebasing token。
- ERC777 callback token。
- `balanceOf`、`transfer`、`transferFrom` 行为不标准的 token。
- Decimals 或转账行为没有经过测试覆盖的 token。

## 风险 1：Oracle Manipulation

### 描述

Vault 使用 TWAP 和 volatility 信号决定流动性应该进入哪个 venue。如果 oracle 被操纵，Vault 可能 rebalance 到错误 venue，或使用不公平的价值计算。

### 缓解措施

- 使用 Uniswap V3 TWAP，而不是 spot price。
- 要求 oracle 已初始化，并且不是 stale。
- 设置最小 TWAP period。
- Rebalance 之间加入 cooldown。
- 当 target venue 不等于 oracle 推荐 venue 时拒绝 rebalance。
- 使用真实 V3 pool 做 fork test。

### 剩余风险

TWAP 只能提高操纵成本，不能完全消除 oracle 风险。低流动性、过短 TWAP 窗口或极端市场条件仍然可能产生不安全信号。

## 风险 2：Reentrancy

### 描述

Vault 和 adapters 会移动 ERC-20 token，并与外部 AMM 合约交互。恶意 token 或异常 callback 可能尝试重入 `deposit`、`withdraw`、`rebalance` 或 `emergencyExit`。

### 缓解措施

- 所有移动资产的 external 函数使用 `nonReentrant`。
- 尽量使用 checks-effects-interactions 顺序。
- 不支持 ERC777 或 callback token。
- Adapter 的 recipient 固定为 Vault。
- 如纳入范围，使用 malicious mock 做重入测试。

### 剩余风险

外部 AMM 集成本身会扩大攻击面。每新增一个 venue adapter，都需要单独审查。

## 风险 3：Slippage 和 Sandwich Attack

### 描述

Rebalance 操作可能被抢跑，或在不利价格下执行。如果 Vault 接受任意 slippage，用户资产可能在移除或添加流动性时损失价值。

### 缓解措施

- 记录 rebalance 前的 Vault 总价值。
- 记录 rebalance 后的 Vault 总价值。
- 如果价值损失超过 `maxSlippageBps`，则 revert。
- 不允许 rebalancer 指定任意 token、adapter 或 recipient。
- 与 V2/V3 交互时使用 minimum amounts。
- 使用 cooldown 降低重复无意义 rebalance 的风险。

### 剩余风险

Slippage 检查可以降低损失，但不能保证最优执行。当前版本不是生产级最优策略。

## 风险 4：Access Control 错误

### 描述

如果权限控制错误，恶意用户可能替换 adapter、修改 oracle、暂停用户提款，或重定向资产。

### 缓解措施

- Adapter、oracle 和风险参数修改使用 `onlyOwner`。
- Rebalancer 只能触发已通过校验的 rebalance。
- Vault paused 时仍允许用户 withdraw。
- 关键配置变化必须 emit event。
- 不暴露可以直接转移用户资产的 admin 函数。

### 剩余风险

Owner 被攻破仍然是严重风险。生产版本应该使用 multisig 和 timelock。

## 风险 5：Rounding 和 Dust

### 描述

Vault shares 会计和按比例提款涉及除法与舍入。反复小额 deposit / withdraw 可能产生 dust 或不公平的舍入效果。

### 缓解措施

- 在 `VaultMath` 中明确舍入方向。
- 当 minted shares 为 0 时 revert。
- Deposit amount 为 0 时 revert。
- 使用 `minShares`、`minAmount0`、`minAmount1` 保护用户。
- 测试小额存款、多用户存款、按比例提款。

### 剩余风险

少量 dust 可能残留在 Vault 或 adapters 中。实现时必须避免 dust 被反复提取。

## Emergency Controls

Vault 应实现：

- `pause()`：禁用 deposit 和 rebalance。
- `unpause()`：恢复正常操作。
- `withdraw()`：paused 状态下仍然可用。
- `emergencyExit()`：从当前 adapter 移除流动性，回到 IDLE，并 pause Vault。

## 部署前假设

Sepolia 部署前应确认：

- Token 地址已确认。
- V2/V3 pool 和 router 地址已确认。
- Oracle period 和 staleness 参数已配置。
- Adapter 地址已配置到 Vault。
- 针对选定 Uniswap 合约的 fork tests 已通过。
- 合约验证成功。

## 审计状态

本项目未经过审计。

请勿存入真实资金。
