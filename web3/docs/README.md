# Web3 深度学习文档索引

本文档汇总了基于您的学习计划生成的深度分析文档。

## 文档列表

### 1. [基础设施与核心原语](01_infrastructure_primitives.md)
*对应计划阶段一*
- **密码学**: Hash (Keccak-256), 签名 (ECDSA), Merkle Tries.
- **共识机制**: PoW (Nakamoto) vs PoS (Gasper).
- **EVM**: 堆栈架构, Gas 机制, 存储布局.

### 2. [DeFi 协议与核心算法](02_defi_mechanisms.md)
*对应计划阶段三*
- **AMM**: Uniswap $x*y=k$ 公式推导与滑点计算.
- **借贷**: 资金池模型与利率曲线.
- **稳定币**: MakerDAO 抵押债仓 (CDP) 与锚定机制.

### 3. [MEV 与投机逻辑](03_mev_and_speculation.md)
*对应计划阶段四与五*
- **MEV**: 抢跑, 夹子攻击, Flashbots 架构.
- **闪电贷**: 原子性借贷套利原理与攻击向量.
- **流动性挖矿**: APY 计算陷阱与投机周期.

### 4. [超越 DeFi 的应用场景](04_beyond_defi.md)
*对应计划阶段四*
- **NFT**: ERC-721/1155 标准, 游戏资产, RWA.
- **DePIN**: 物理基础设施网络 (存储, 计算).
- **DeSoc**: 社交图谱上链与创作者经济.
- **DAO**: 链上治理与智能合约执行.

## 下一步建议

完成上述文档的阅读后，建议按照原计划进入 **阶段二：开发与智能合约**。
虽然您暂时搁置了编码任务，但理解代码（即使不写）对于深入掌握上述概念至关重要。

推荐后续步骤：
1. 阅读 Solidity 官方文档中关于 Security 的章节。
2. 在 Etherscan 上查看真实的 Uniswap Router 合约代码，对照文档中的数学公式。

