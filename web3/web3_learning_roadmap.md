# Web3 全栈技术与经济模型深度学习计划

这份计划结合您的计算机科班背景，旨在快速掌握 Web3 核心技术。我们将从底层原理（操作系统/网络/密码学）切入，逐步深入到智能合约开发、架构模式，最后解析经济模型与投机逻辑。

## 阶段一：基础设施与核心原语 (CS 视角)

> **[详细文档: 基础设施与核心原语](docs/01_infrastructure_primitives.md)**

**目标：** 理解“状态机”的底层运作机制。

- **密码学基础 (Cryptography):**
    - 哈希函数：Keccak-256 (SHA-3 家族) 原理与抗碰撞性。
    - 公钥加密：ECDSA (secp256k1) vs EdDSA。签名与验证流程。
    - 进阶：Merkle Trees (成员证明), Bloom Filters (日志检索), 零知识证明 (概念：在不泄露数据的前提下证明计算的正确性)。
- **分布式系统 (Distributed Systems):**
    - 共识机制：Nakamoto Consensus (PoW), Gasper (ETH PoS - LMD GHOST + Casper FFG)。
    - 网络层：P2P Gossip 协议, 节点发现 (Kademlia DHT)。
    - 存储层：日志结构存储, LevelDB/RocksDB 在节点中的应用。
- **以太坊虚拟机 (EVM):**
    - 架构：基于堆栈的虚拟机, 256位字长。
    - 状态：世界状态树 (Patricia Merkle Trie)。
    - 执行：Opcode 指令集, Gas 计费机制 (停机问题解法), 内存布局 (Memory vs Storage vs Stack vs Calldata)。

## 阶段二：开发与智能合约

**目标：** 编写安全、高效的链上逻辑。

- **Solidity (编程语言):**
    - 语法与类型：值类型, 映射 (Mappings), 结构体 (Structs)。
    - 安全模式：Checks-Effects-Interactions, 防重入 (Reentrancy Guards), 代理模式 (升级机制)。
    - Gas 优化：变量打包, 内存管理, Assembly (Yul) 低级控制。
- **工具链 (开发环境):**
    - **Foundry (基于 Rust):** 推荐 CS 专业人士使用。极速测试, Fuzzing (模糊测试), 详细堆栈追踪。
    - Hardhat (JS/TS): 传统主流, 插件生态丰富。
    - 客户端交互: Viem / Ethers.js (连接前端与区块链)。

## 阶段三：核心架构与著名产品

**目标：** 理解业界标准模式及其解决的问题。

### 1. DeFi (去中心化金融) - 金融技术栈

> **[详细文档: DeFi 协议与核心算法](docs/02_defi_mechanisms.md)**

- **DEX (去中心化交易所):**
    - *代表产品:* **Uniswap**。
    - *核心概念:* **AMM (自动做市商)**，基于 `x * y = k` 公式。替代了传统订单薄。
    - *CS 问题解决:* 链上异步执行导致订单薄成本过高；AMM 提供了无需许可的连续流动性。
- **Lending (借贷市场):**
    - *代表产品:* **Aave, Compound**。
    - *核心概念:* 资金池模式 + 超额抵押。
    - *CS/经济逻辑:* 基于资金利用率的算法利率 (供需曲线编码在合约中)。
- **Stablecoins (稳定币):**
    - *代表产品:* **MakerDAO (DAI)**, **USDC**。
    - *核心概念:* 通过激励机制或资产储备实现价值软锚定。
    - *机制:* CDP (债仓) 与清算机器人 (Keeper)。

### 2. 基础设施 (Infrastructure)

- **Oracles (预言机):**
    - *代表产品:* **Chainlink**。
    - *解决问题:* 区块链是确定性的封闭系统，无法主动获取外部 API 数据。预言机作为可信中间件引入数据。
- **Bridges (跨链桥):**
    - *代表产品:* **LayerZero, Wormhole**。
    - *解决问题:* 跨链状态同步与消息传递。

### 3. 多元化应用场景 (NFT, DAO, Social)

> **[详细文档: Web3 的多元化应用场景](docs/04_beyond_defi.md)**

- **NFT & Digital Ownership:**
    - *产品:* **OpenSea (Marketplace)**, **BAYC (IP)**.
    - *解决问题:* 数字资产的确权 (ERC-721/1155)。
- **DeSoc (去中心化社交):**
    - *产品:* **Lens Protocol, Farcaster**.
    - *解决问题:* 用户拥有数据所有权，抗审查的社交图谱。
- **DePIN (去中心化物理基建):**
    - *产品:* **Filecoin (存储), Helium (网络)**.
    - *解决问题:* 众包模式建设物理基础设施。

### 4. 扩容方案 (Layer 2)

- **Rollups:**
    - *核心概念:* 链下执行计算，链上发布数据。
    - *Optimistic (Arbitrum):* 乐观假设交易有效，通过欺诈证明 (Fraud Proof) 挑战。
    - *ZK (Starknet/zkSync):* 提交密码学有效性证明 (SNARK/STARK) 验证状态。

## 阶段四：价值捕获与投机逻辑

> **[详细文档: MEV 与投机逻辑](docs/03_mev_and_speculation.md)**

**目标：** 客观分析“黑暗森林”中的博弈。

### 1. MEV (最大可提取价值) - *高阶套利*

- **逻辑:** 矿工/排序器有权决定区块内交易的顺序。
- **技术手段:**
    - *抢跑 (Front-running):* 监控内存池 (Mempool) 中的大额买单，在它之前买入，随后高价卖出。
    - *夹子 (Sandwiching):* 在受害者买入前买入，受害者买入推高价格后立即卖出。
- **技术栈:** 修改版节点客户端 (Geth), Flashbots bundles (隐私交易池)。

### 2. 闪电贷与原子套利 (Flash Loans)

- **逻辑:** 无需抵押即可借出数百万美元，前提是资金必须在**同一笔交易**内归还。
- **应用:** DEX 间的无风险搬砖套利。如果套利失败，交易回滚（借款从未发生），仅损失 Gas 费。

### 3. 流动性挖矿 (Yield Farming)

- **逻辑:** 协议补贴流动性成本。协议向存入资产的用户分发治理代币。
- **投机:** 赚取高额 APY (通常 >100%)，赌代币升值收益超过“无常损失” (Impermanent Loss)。

### 4. 撸空投 (Airdrop Farming)

- **逻辑:** 早期协议给予测试用户的追溯性奖励。
- **策略:** 编写脚本自动化交互 (Swap, Bridge)，覆盖大量钱包地址。
- **对抗:** 女巫攻击检测 (Sybil detection) —— 通过资金来源图谱或行为聚类剔除刷单者。

## 执行计划

1. **环境准备:** 安装 Rust, Foundry, Node.js。
2. **构建实践:** 编写一个简单的 AMM (Uniswap 克隆版) 和 质押 (Staking) 合约。
3. **深入分析:** 阅读 Bitcoin, Ethereum 和 Uniswap V2 的白皮书。
4. **实验:** 编写脚本监听 Mempool 中的大额交易。

