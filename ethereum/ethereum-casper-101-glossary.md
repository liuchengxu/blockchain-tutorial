以太坊 Casper 101
=================

> 原文: [Ethereum Casper 101](https://medium.com/@jonchoi/ethereum-casper-101-7a851a4f1eb0)

## 词汇表

[**Proof of Stake**](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ) -- 公有链的一类共识算法，基于验证人（validator）在网络中的经济权益（economic stake）。

[**Casper**](https://blog.ethereum.org/2015/08/01/introducing-casper-friendly-ghost/) -- 以太坊的 PoS 研究和项目。

**Finality** - 系统中的操作一旦完成，就永远再无法回滚（[Vitalik on settlement finality](https://medium.com/@jonchoi/an%20attack%20wherein%20a%20reputation%20system%20is%20subverted%20by%20forging%20identities%20in%20peer-to-peer%20networks)）背景：在 PoW 中，finality（最终确定性） 是概率性且隐式的。Casper 所设计的机制，是显式强制保证最终确定性。

[**Fork Choice Rule**](https://medium.com/@VitalikButerin/minimal-slashing-conditions-20f0b500fc6c) - 分叉选择规则（fork choice rule）就是一个函数，由客户端进行评估，输入为区块或是其他生成的信息，向客户端输出“标准链（canonical chain）”是什么。

[**Slashing Conditions**](https://medium.com/@VitalikButerin/minimal-slashing-conditions-20f0b500fc6c) - 通过在 P2P 网络中伪造身份，进而破坏一个信誉系统（reputation system）的攻击。

[**3 E’s of Sybil Resistance**](https://twitter.com/dominic_w/status/648330685963370496) - 1. 进入成本（entry cost）2. 存在成本（existence cost）3. 退出惩罚（exit penalty）。（源自 [Dominic Williams](https://medium.com/@dominic_w)）。

[**Nothing-at-stake problem**](https://ethereum.stackexchange.com/questions/2402/what-exactly-is-the-nothing-at-stake-problem) - 一个 PoS 实现上的挑战，它指的是如果出现分叉，无法同时验证两条链。这是 PoS 的一个老大难问题，但是通常也被认为是可解决的。比如，参见 [Slasher](https://blog.ethereum.org/2014/01/15/slasher-a-punitive-proof-of-stake-algorithm/)。

**Bribe attack（贿赂攻击）** - 攻击者用贿赂的方式，改变验证人博弈论框架的纳什均衡，以此来削弱协议的安全性。（ 更多背景见 [History of Capser pt 2](https://blog.ethereum.org/2016/12/07/history-casper-chapter-2/)）

[**Long range attack（长程攻击）**](https://blog.ethereum.org/2014/05/15/long-range-attacks-the-serious-problem-with-adaptive-proof-of-work/) - 与 51% 攻击（为了攻击者的利益让一个更长的链重写账本）的机制一样，但并不是发起 6 个块的攻击，而是攻击更多块（比如 60,000 个块）。

[**DAG**](https://en.wikipedia.org/wiki/Directed_acyclic_graph) - “有向无环图（Directed Acyclic Graph）”。一个没有有向环的有限有向图。（[ETH Stack Exchange](https://ethereum.stackexchange.com/questions/1993/what-actually-is-a-dag)）

**GHOST ** - “Greedy Heaviest Observed Subtree”. 它是一个链的选择规则，目的是快速确认，同时尽量减少在安全性或去中心化上的妥协。（[原始论文](https://eprint.iacr.org/2013/881.pdf)，[ETH GHOST](https://github.com/ethereum/wiki/wiki/White-Paper#modified-ghost-implementation)）

**Synchronicity** - 指的是消息的时间假设（也就是说，同步，部分同步还是异步）。

**Liveness** - “availability（可用性）”。遵守协议的节点最终决定出块。其反面就是网络状态会因为出块决定而阻塞（也就是说，没有 2/3 投票高度的 Tendermint）

**Safety** - “correctness”。遵守协议的节点会就出块达成共识。另一个直观解释是两个冲突的块是否可以提交。

[**FLP Impossibility Theorem（FLP 不可能定理）**](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) - “不可能有一个 live，safe，asynchronous 的网络”（已被证明）

**Accountable Faults** - 由于某一个验证人，或者某一些验证人引起的错误。

[**Byzantine Faults（拜占庭错误）**](https://en.wikipedia.org/wiki/Byzantine_fault_tolerance)/Byzantine Behavior -- 对于对于不同观察者显示出不同症状的任意错误。非协议遵守行为。

**Byzantine Failure（拜占庭故障）** -- 在一个需要 [consensus](https://en.wikipedia.org/wiki/Consensus_%28computer_science%29) 的系统中，由于拜占庭错误引发的系统服务损失。

[**Byzantine Fault Tolerance**](https://en.wikipedia.org/wiki/Byzantine_fault_tolerance)（“BFT”）-- 一个系统能够容忍拜占庭错误的能力。1/3 的拜占庭错误阈值在异步网络中。1/2 在同步网络中。（BFT 共识算法包括 Paxos，PBFT 和 Casper ，Tendermint）.

**Nakamoto Consensus** -- 类似比特币就 PoW 的共识。此外，Nakamoto 风格的共识存在，将会是基于链的 PoS 而不是基于 BFT 的 PoS。

[**Tendermint**](https://tendermint.com/static/docs/tendermint.pdf) -- 专注一致性（consistency）的 PoS 实现。在少于 1/3 恶意参与者的情况下，永远不会分叉，但是缺点是，如果缺乏 2/3 的验证人投票，链可能会停止运转。

**Validator（验证人）** -- 一个为了获取奖励而验证 checkpoint 或是区块的 entity，是 PoS 中的 “矿工”。

**Validator Set（验证人集合）** -- 一条链在任意给定时间的验证人集合。

**Checkpoint** -- 在 FFG 中，它是按照规律（比如每 50 个块）隔开的一个块，这个块在底层是 PoW 的链之上，同时采用 PoS 验证机制（比如 Ethereum with ethash）。

**Epoch** -- 在 FFG 中，50 个块为一个周期，验证人可以对周期的最后一个块（也就是 checkpoint）进行投票。PoW 矿工出块，PoS 验证人在每个周期对 checkpoint 进行验证。

**Dynamic Validator Sets（动态验证人集合）** -- 在一个时期内，一条链可以有一个变化的验证人集合。在 BFT 风格的共识算法中，这是一个巨大的改进。Tendermint 是第一个突破，Casper 也正在这一点上积极运作。

**Equivocation** -- 验证人发送两个相互冲突的消息（ 更准确的定义见 [slide 28 of this deck](https://ethereumfoundation.org/devcon2/wp-content/uploads/2016/10/A-Correct-by-Construction-Asynchronous-Casper-Protocol.pdf)）

**Dunkles** -- 包含从非主导（non-dominant）块向主导（dominant）块数据的机制。它提供了一个更好的激励机制，可以有效缓解 nothing-at-stake 问题（[link](https://ethereum.stackexchange.com/questions/19146/are-there-still-uncles-with-proof-of-stake)）。

**Proposal Mechanism（提案机制）** -- 集合中的验证人提议哪个块进行公正（justification）或最终确定（finalization）的机制。

**Justification（公正）** -- 比如在 FFG 中，如果验证人集合中有 2/3 的人对一个新的 checkpoint 投票，那么该 checkpoint 就是准确的记录。这是形成一个最终确定化（finalized）的 checkpoint 的中间过程.

**Finalization（最终确定）** -- 比如在 FFG 中，如果 2/3 的验证人对一个被公正过的 checkpoint 投票，那么这个 checkpoint 就是准确的记录。这一步完成后，checkpoint 被最终确定。

[**State Transition System（状态转移系统）**](https://github.com/ethereum/wiki/wiki/White-Paper#bitcoin-as-a-state-transition-system) -- 一个维护给定状态（比如交易或账户集合）和它随着时间（也就是转移，transition）变化的系统。比特币，以太坊和其他公有链可以被认为是状态转移系统。

**Protocol Utility Function（协议效用函数）** -- 一个告诉我们协议当前执行状况的公式，理想状态下应该是从区块链内部进行计算的（calculable from inside the blockchain）。在 PoW 的链中，它可以是所有生成的区块占主链的百分比。在 Casper 中，如果协议效用（protocol utility）是 0，表示执行完美，每个 epoch 都被最终确定，也没有发生安全故障。对每个没有最终确定的 epoch，会有一些惩罚，而对于每个安全故障会有一个非常严重的惩罚。如果一个协议效用函数可以被形式化，那么故障惩罚就可以尽可能与这些故障所造成的协议损失相匹配。（来自 [Triangle of Harm](http://vitalik.ca/general/2017/07/16/triangle_of_harm.html)）
