# 深入解析 Web3 基础设施与核心原语

本文档从计算机科学角度深入剖析 Web3 的底层构建模块，对应学习计划的 **阶段一**。

## 1. 密码学基础 (Cryptography)

Web3 的“信任”建立在密码学原语之上，而非中心化机构。

### 1.1 哈希函数 (Hash Functions)
- **核心算法**: Ethereum 使用 **Keccak-256** (SHA-3 的前身)，而非标准的 SHA-256 (Bitcoin 使用)。
- **特性**:
    - **抗碰撞性 (Collision Resistance)**: 难以找到两个不同的输入 $x$ 和 $y$，使得 $H(x) = H(y)$。
    - **雪崩效应 (Avalanche Effect)**: 输入微小的改变会导致输出巨大的差异。
- **应用**:
    - **地址生成**: 公钥的哈希值的后 20 字节。
    - **数据指纹**: 交易 ID (Tx Hash)、区块哈希。
    - **承诺 (Commitment)**: 在不揭示原始内容的情况下锁定数值。

### 1.2 数字签名 (Digital Signatures)
- **核心算法**: **ECDSA** (Elliptic Curve Digital Signature Algorithm) over **secp256k1** 曲线。
    - 这一点与常见的 NIST 曲线 (secp256r1) 不同，Bitcoin 选择了 Koblitz 曲线，Ethereum 沿用了这一选择。
- **流程**:
    1.  **私钥 (Private Key)**: 随机生成的 256 位整数 $k$。
    2.  **公钥 (Public Key)**: $K = k \times G$ (其中 $G$ 是基点，$\times$ 是椭圆曲线乘法)。这是一个单向陷门函数。
    3.  **签名**: 使用私钥 $k$ 对消息哈希 $z$ 进行签名，生成 $(r, s)$ 对。
    4.  **验证**: 任何人可以使用公钥 $K$ 验证 $(r, s)$ 是否由持有 $k$ 的人生成，且消息 $z$ 未被篡改。
    - **注意**: 严禁重复使用随机数 $k$ (nonce)，否则会导致私钥泄露。

### 1.3 Merkle Trees & Merkle Patricia Tries
- **Merkle Tree**:
    - 二叉树结构，叶子节点是数据块的哈希，非叶子节点是子节点哈希的哈希。
    - **作用**: 快速验证数据存在性 (Merkle Proof)，只需 $O(\log n)$ 的路径哈希。
- **Merkle Patricia Trie (MPT)**:
    - Ethereum 的核心数据结构，结合了 Radix Trie (压缩前缀树) 和 Merkle Tree。
    - **状态存储**: 存储所有账户的状态 (Nonce, Balance, StorageRoot, CodeHash)。
    - **特性**: 它是确定性的 (Deterministic)，相同的状态集合必然生成相同的 Root Hash。任何状态的微小改变都会导致 Root Hash 改变。

---

## 2. 分布式系统与共识 (Consensus)

解决的核心问题是：在没有中央协调者的情况下，如何让分布式网络对“状态”达成一致。

### 2.1 Nakamoto Consensus (PoW - Bitcoin/Eth 1.0)
- **机制**: 最长链原则 (Longest Chain Rule) + 工作量证明 (Proof of Work)。
- **Sybil Resistance**: 通过消耗算力 (CPU/ASIC) 来阻止女巫攻击。
- **Finality (最终性)**: 概率性最终。随着确认区块数的增加，交易被回滚的概率指数级下降。

### 2.2 Gasper (Ethereum 2.0 / PoS)
Ethereum 目前使用的共识机制是 **Gasper**，它是 **LMD-GHOST** 和 **Casper FFG** 的结合。
- **LMD-GHOST (Latest Message Driven Greedy Heaviest Observed Subtree)**:
    - 用于**区块选择 (Fork Choice Rule)**。节点总是选择“投票”权重最高的子树作为主链。这保证了链的活性 (Liveness)。
- **Casper FFG (Friendly Finality Gadget)**:
    - 用于**最终性 (Finality)**。
    - **Epoch**: 每 32 个 Slot (约 6.4 分钟) 为一个 Epoch。
    - **Justification & Finalization**: 验证者对 Epoch 进行投票。当 2/3 的验证者同意时，Epoch 变为 Justified，随后变为 Finalized。
    - **Slashing (罚没)**: 如果验证者表现恶意 (如双重投票)，其质押的 32 ETH 会被罚没。这是 PoS 区别于 PoW 的关键经济惩罚机制。

---

## 3. 以太坊虚拟机 (EVM)

EVM 是一个准图灵完备的、基于堆栈的虚拟机。

### 3.1 架构
- **Stack (堆栈)**: 256 位宽度，最大深度 1024。主要用于计算。
- **Memory (内存)**: 线性字节数组，易失性 (交易结束后清除)。随着使用量扩展，Gas 成本呈二次方增长。
- **Storage (存储)**: 键值对映射 (256-bit key -> 256-bit value)，持久性 (写入区块链状态)。**最昂贵**的操作 (SSTORE)。
- **Calldata**: 存放交易输入数据的只读区域。

### 3.2 为什么是 256 位？
- 便于进行密码学运算 (如 Keccak-256 哈希, 椭圆曲线运算)。
- 缺点: 对于普通的数学运算 (如 64 位整数加法)，效率低于原生 CPU 指令。

### 3.3 Gas 机制
- **停机问题 (Halting Problem)**: 图灵完备系统无法静态判断程序是否会无限循环。
- **解决方案**: 每条指令 (Opcode) 都有固定的 Gas 成本。交易必须携带预付的 Gas。如果 Gas 耗尽，EVM 抛出 OutOfGas 异常，**状态回滚**，但**费用不退**。

