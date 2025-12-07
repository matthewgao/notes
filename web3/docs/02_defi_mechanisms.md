# 深入解析 DeFi 协议与核心算法

本文档深入解析去中心化金融 (DeFi) 的数学模型与核心机制，对应学习计划的 **阶段三**。

## 1. 自动做市商 (AMM) - 以 Uniswap V2 为例

传统的中心化交易所 (CEX) 使用**中央限价订单簿 (CLOB)**，依赖做市商提供流动性。链上存储和匹配订单极其昂贵，因此诞生了 AMM。

### 1.1 恒定乘积公式 (Constant Product Formula)
核心公式：
$$x \cdot y = k$$

- $x$: 流动性池中 Token A 的数量
- $y$: 流动性池中 Token B 的数量
- $k$: 恒定常数 (在单次交易中保持不变，但在添加/移除流动性时改变)

### 1.2 价格发现
在 AMM 中，价格不是由预言机决定的，而是由池中两种资产的比率决定的。
- Token A 对 Token B 的价格 $P_A = \frac{y}{x}$
- Token B 对 Token A 的价格 $P_B = \frac{x}{y}$

### 1.3 交易滑点 (Slippage) 计算
假设用户想用 $\Delta x$ 个 Token A 购买 Token B，能买到多少 $\Delta y$？

根据公式：
$$(x + \Delta x)(y - \Delta y) = k = x \cdot y$$

推导 $\Delta y$:
$$y - \Delta y = \frac{xy}{x + \Delta x}$$
$$\Delta y = y - \frac{xy}{x + \Delta x} = \frac{y(x + \Delta x) - xy}{x + \Delta x} = \frac{y \Delta x}{x + \Delta x}$$

考虑到手续费 (Uniswap V2 为 0.3%)，实际输入的 $\Delta x$ 变为 $\Delta x \cdot 0.997$。

### 1.4 无常损失 (Impermanent Loss - IL)
这是流动性提供者 (LP) 面临的主要风险。当池内代币价格相对于存入时发生变化，LP 在撤资时获得的资产总价值，会低于“如果不做 LP 而是直接持有 (HODL)”的价值。
- **本质**: AMM 机制迫使 LP 在价格上涨时卖出资产，在价格下跌时买入资产（类似于“高买低卖”的反向操作）。
- 这种损失被称为“无常”，是因为如果价格回归到初始比率，损失会消失。但如果 LP 在价格偏离时退出，损失就变成“永久”的。

---

## 2. 去中心化借贷 (Lending) - 以 Aave/Compound 为例

### 2.1 核心逻辑：超额抵押 (Over-Collateralization)
由于区块链上没有信用体系，所有借贷必须是超额抵押的。
- **LTV (Loan-to-Value)**: 贷款价值比。例如 LTV=75%，意味着抵押 100 USD 的 ETH，最多借出 75 USD 的 USDC。
- **清算阈值 (Liquidation Threshold)**: 如果抵押品价值下跌，导致债务/抵押品价值 > 阈值，清算人 (Liquidator) 可以触发清算，替借款人偿还部分债务，并以折扣价获得其抵押品。

### 2.2 资金池模型 (Pooled Liquidity)
用户不直接借给另一个用户（点对点），而是借给协议的智能合约（点对池）。

### 2.3 利率模型 (Interest Rate Model)
利率 $R$ 是资金利用率 $U$ (Utilization Rate) 的函数。
$$U = \frac{\text{Total Borrows}}{\text{Total Liquidity}}$$

通常采用**分段线性函数 (Kinked Rate Model)**:
- 当 $U < U_{optimal}$ (最佳利用率，如 80%) 时，利率缓慢增长，鼓励借款。
- 当 $U > U_{optimal}$ 时，利率急剧飙升，迫使借款人还款或吸引更多存款人，以避免流动性枯竭。

---

## 3. 稳定币 (Stablecoins) - 以 MakerDAO (DAI) 为例

### 3.1 抵押债仓 (CDP / Vault)
用户存入 ETH 等波动资产，生成 (Mint) 稳定币 DAI。这本质上是一笔抵押贷款。

### 3.2 锚定机制 (Peg Stability)
- **当 DAI < $1**: 市场上 DAI 供过于求。用户可以低价买入 DAI，偿还债务（只需 $0.98 还 $1 的债），销毁 DAI，减少供给，推高价格。
- **当 DAI > $1**: 市场上 DAI 供不应求。用户会开启新 Vault，生成更多 DAI 抛售获利，增加供给，压低价格。
- **PSM (Peg Stability Module)**: 允许用户以 1:1 的汇率用 USDC 铸造 DAI，利用套利将价格硬锁定在 $1。

### 3.3 风险参数
- **Stability Fee**: 贷款利息，用于调控 DAI 的供需。
- **Liquidation Ratio**: 清算线。如果 ETH 大跌，Vault 会被强制拍卖。

