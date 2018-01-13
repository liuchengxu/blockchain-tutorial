以太坊 PoS 概览：Casper FFG
===========================

>出处: [Ethereum PoS Overview: Casper FFG](https://docs.google.com/presentation/d/1fqnjL-2TqXjhHx8k7HRX7eUYnDK83adnlCLLH8Bk054/edit#slide=id.g29703948a2_0_2965)

简单介绍了区块链概念、工作量证明协议的工作原理，以及工作量证明的缺点。

## 2. PoW 的缺点

1. 浪费资源。

2. 易受 ASIC 和中心化的攻击

3. “最终确定性（finality）” 不足

介绍 Casper FFG 的基本概念和原理，如：权益、检查点、投票以及最终化。

### "Final Gadget" 是什么

Casper the Friendly Finality Gadget 是以太坊上 PoS 的第一个部署阶段，主要内容有：

1. 在 PoW 之上加了一层 PoS
2. 每 50 个块（一个 epoch）实现终态(finality)
3. 抵抗 51% 攻击
4. 相对于 PoW 减少能源浪费

在后续的迭代升级中，PoW 会被完全替代。

### 验证人

#### 成为验证人

任何 ETH 的持有者只要将 ETH 存入 Casper 智能合约，就可以成为验证人。

Casper 的激励逻辑存在于合约内。

#### 为什么收取保证金

- 给“好人”以更大的经济激励
- 给“坏人”以更大的经济惩罚

通过保证金强制惩罚恶意参与者。

整条链以 50 个块进行分隔，叫做一个 epoch，第 50 个块为 checkpoint。

当两个连续的 checkpoint 得到 2/3 的投票，就最终确定，不可修改。justified checkpoint.


介绍 Casper FFG 的惩罚条件。

介绍 Casper FFG 的经济激励。

介绍第五章中的经济激励机制如何能够遏制恶意攻击，以及 Casper 协议下一步的方向。
