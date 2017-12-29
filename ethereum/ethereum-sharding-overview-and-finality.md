以太坊分片：overview and finality
=================================

在 [Ethereum Casper 101](https://medium.com/@jonchoi/ethereum-casper-101-7a851a4f1eb0)[1] 中，Jon Choi 对 Casper 做了一个很棒很清晰的综述，并解释了*为什么*显式最终确定性（explicit finality）对于可扩展性（scalability）大有裨益。本文旨在给出一个以太坊分片的设计概览，并阐释显式最终确定性*如何*有助于区块链分片。

为了完全理解以太坊分片机制提案的技术规范，我强烈推荐深入研究 Vitalik 写的  [**sharding doc**](https://github.com/ethereum/sharding/blob/develop/docs/doc.md) [2].

## 区块链可扩展性问题

1. **不断增长的交易**。

2. **目前的块生成过程导致可扩展性受限**。区块的 *gas limit* 束缚了区块的计算容量。无论是提高区块的 gas 上限，还是大大降低区块时间，都会导致高陈腐率（high stale rate），并削弱网络对抗攻击的能力。

3. **并行不足**。首先，现有的 EVM 按先后顺序依次处理交易。其次，出于安全和去中心化的考虑，每个全节点会执行每一笔交易，并存储整个（或修剪后）的状态树。

>进阶阅读：并行执行交易 [EIP 648 — Easy parallelizability](https://github.com/ethereum/EIPs/issues/648)

为了解决可扩展性问题，分片（sharding）引入了链上状态分区（on-chain state partition）来获得更高的吞吐量。


## 术语

首先，让我们来看一下在主链（你可以理解为现在的 Mainnet chain）和分片链（shard chain）上不同层次的对象区别：

![Table 1. Terminology.](http://upload-images.jianshu.io/upload_images/127313-8d156e10bade5a17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以简单地这么认为，交易都会被装入 “collation”。与区块类似，一个 collation 也会指向它在链（指的是分片链）上的 parent collation。 成为一个 “collator”，就意味着你有资格在 POS 分片链上提名一个新的 collation。

![Figure 1. A glimpse of basic collation data structure.](http://upload-images.jianshu.io/upload_images/127313-3a69614f0c028914.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 基本的二次分片

### 分片链的共识依赖于主链

与侧链类似，collation 只有**一小部分的证明**必须记录在主链上 --  这也是我们如何扩展区块链的基本想法：

1. 分片链上的交易处于自己独立的空间中，分片验证人（shard validator）*只*需要验证他们所关注的分片。

2. 分片链也通过 POS 机制依附于主链，以获得 *更高层次的共识（higher level of consensus）*。

### 验证人管理员合约（Validator Manager Contract, VMC）

为了将分片链加入到主链中，在主链上需要有一个叫做验证人管理员合约（VMC）的特殊合约。VMC 是这个分片机制的核心。VMC 的目的可以概括如下：

1. **权益证明系统**。如果验证者表现不端，其权益将会被剥削。

2. **伪随机采样**。通过将当前块哈希作为种子，采样出合格的 collator。基本上，验证者将它们的保证金（stake）存入 VMC，然后他们的验证代码地址（validation code address）将会被记录在一个 VMC 内部的 **全局验证人列表（a global validators pool list）**。系统将会从验证人列表中采样出一个分片链的验证人，并将其指定为指定**“时期（period，下面会解释什么是 period）”**内，指定分片的验证人。这种方式使得验证者无法提前预测他们何时会成为验证者，也无法预测会成为哪个分片的验证人。

3. **Collation header 验证**。VMC 有一个 `addHeader(bytes collationHeader)` 函数，该函数用=来验证 collation header，并记录有效的 collation  header hash。这个函数提供了即时的 **链上** 验证。

4. **跨分片通信（cross-shard communication）**。利用 UTXO 模型，并通过在主链上进行交易和创建一个 receipt（带有 receipt ID），用户可以将以太存入一个指定分片。分片链上的用户可以给定 receipt ID 创建一个消费 receipt（receipt-consuming）的交易，来花费该 receipt。

5. **链上治理（on-chain governance）**。将 VMC 作为议会，使得验证人可以在链上进行投票。

## 如何在分片内提名 Collation？

在阶段 1，VMC 将会维护 100 个分片（`SHARD_COUNT = 100`）。每个分片**并行**执行，分片 *i* 的客户端只需要验证分片 *i* 上的交易。

“时期（period）”被定义为一个区块时间的准备窗口（a bounding a window of block times），比如 `PERIOD_LENGTH = 5` 意味着每个周期有 5 个块。这表明在每个周期内，对于每个分片只有不超过 **1** 个有效的 collation。

![Figure 2 (a). Quadratic sharding. The proofs of shard states would be recorded on main chain VMC.](http://upload-images.jianshu.io/upload_images/127313-48dab933ae6ea057.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一旦验证人被采样为合格的 collator 来提案一个新的 collation，collator 必须对最近的 collation 进行验证，并发送一笔交易来调用 `addHeader` 函数。注意，如果 collator **周期 10** 被采样到提交一个新的 collation，这意味着 `addHeader` 交易 **必须被包含在周期 10 里面**，也就是说，交易必须在区块号 `10 * PERIOD_LENGTH` 到区块号 `(10 + 1) * PERIOD_LENGTH - 1` 之间.

![Figure 2 (b). For one shard, only one collation per period; one block can include multiple addHeader transactions of different shards.](http://upload-images.jianshu.io/upload_images/127313-557a7b4ee36264f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

collation header hash 必须被记录在 VMC 上，以证明它的 header 全局有效。此外，分片的所有其他验证人必须时刻检测 VMC 以获得最新状态，然后验证交易是否也有效。

## 分片链的分叉选择规则（fork choice rule）

在基本分片中，分叉选择规则依赖于最长主链。给定一个分片，它的有效 head collation **不是**简单的“最长有效分片链（longest valid shard chain）”的 head collation，而是“**在最长有效主链里面的最长有效分片链（the longest valid shard chain within the longest valid main chain）**”。

Figure 3(a) 中有个例子，主链上有两个分叉，在下图中第二条链是最长有效主链。因为 `block B3` 是 head block，很容易看出 `collation C3` 是 head collation。

![Figure 3 (a).](http://upload-images.jianshu.io/upload_images/127313-2130de02ce98ecbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后 Figure 3(b) 中 `block B3'` 到了。假设 `block B3` 的得分（score）高于 `block B3'`，那么上面的链仍然是最长主链：

![Figure 2 (b).](http://upload-images.jianshu.io/upload_images/127313-00906f6529d95f84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后 Figure 3(c) 到了 `block 4`。注意到，对于这个分片，虽然 `collation C3` 的得分比 `collation C2` 更高，但是下方的链是**最长有效主链**，所有现在 `collation C2` 是 head collation：

![Figure 3 (c)](http://upload-images.jianshu.io/upload_images/127313-63cf06f07825bec7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>更多内容：另一个设计 -- Vlad Zamfir 的 [sharded fork choice rule](https://twitter.com/VladZamfir/status/945358660187893761)

![An ingenious design for guaranteeing blocks atomicity before they are finalized.](http://upload-images.jianshu.io/upload_images/127313-6f6c93920b44ffa1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 可扩展性与安全性之权衡

对于 **去中心化**，**可扩展性** 和 **安全性** 这三个属性，区块链系统最多只能三选其二。

*— Blockchain Trilemma* in [Sharding FAQ](https://github.com/ethereum/wiki/wiki/Sharding-FAQ#this-sounds-like-theres-some-kind-of-scalability-trilemma-at-play-what-is-this-trilemma-and-can-we-break-through-it) [3]

出于对系统安全的保证导致了可扩展性受限 [3]。当为了提高 TPS（每秒交易数）将交易分配到各分片的同时，我们随之也减少了每笔交易的计算资源。

分片的其中一个重要机制就是，**如何在链上生成随机数**。

- collator 被选中的几率，应该仅与验证者的保证金相关，且成比例。

- 如果验证人能够预测，或是任意选择他们想要参与的分片，那么不诚实的验证人既可以相互共谋，展开一个适应性攻击（adaptive attack）。

如果采样不能以较高的随机性进行选择，那么攻击者很可能在分片中展开 **1% 攻击**：如果有 100 个分片，攻击者可以专注于攻击某*一个*分片，他们只需要 1% 的hash rate(POW)/deposit(POS) 就可以控制分片[4].

![Figure 4. Traditional majority attack (51 % Attack)](http://upload-images.jianshu.io/upload_images/127313-9138ceb26bc263b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Figure 5. Sharding 1% attack](http://upload-images.jianshu.io/upload_images/127313-94c92c07edce0330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 分片的区块链显式最终确定性

### 隐式最终确定性 v.s. 显式最终确定性

首先，我必须声明，分片机制应该同时能够应用于 POW 与 POS 的链。即使如此，显式最终确定性这个小物件也跟 Casper 一样，可以使得分片更加健壮。

在一般的 POW 链中，最终确定性是概率性，隐式的[1][5]。简单来说，即使区块获得了数以千次的确认，仍有可能对链进行重写。相反，将 [Casper the Friendly Finality Gadget (“FFG”)](https://github.com/ethereum/research/blob/master/papers/casper-basics/casper_basics.pdf) 加密经济机制应用于 POS ，显式地在协议内（in-protocol）强制保证对我们来说，是否是最终确定 (we-can-check-if-its-finalized-for-us)。

[来自 Vlad] 协议内显式最终确定性阈值有个经济风险：它在 `2/3 + 1` 和 `1/3 + 1` 之间创造了一个理想的 卡特尔大小（cartel size）。相应地，任何不在 `2/3 + 1` 同盟节点（coalition），其最终确定性的边际贡献就将为 0。

### 主链最终确定性依赖

在基本分片中，分片链锚定与主链之上。

对于分片验证人，我们希望分片，区块链容量在阶段 1 扩容 100 倍，因此这 100 个分片的所有验证人，都将需要监测 VMC 状态来获得正确有效的 head collation。对于验证人来说，重要的一点是，要尽快确信他们是否是 collator。对于普通用户而言，如果我们在阶段 2 应用跨分片交易，普通用户也将需要在 VMC 上检索他们的保证金信息（*receipt ID*）。

显式最终确定性，将会有助于缓解主链与大量分片链之间同步的不确定性。

### 显式最终确定性有助于无状态客户端

无状态客户端的基本原则是，它不存储整个状态树，相反，无状态客户端**只存储状态树根**。归档客户端（archival clients）存储整个状态树，并提供给定 collation **所需** 的 Merkle 分支。有了这些 Merkle 分支，无状态客户端就能够构建**部分的状态树**，并验证 collation [6]。

一旦完成验证人采样并再混洗（reshuffle），就会立刻触发同步。有了无状态客户端机制，**再混洗**（也就是改变验证者检测的分片，并同步分片链）的成本低至（接近）0，因为它们**只需要验证最新的 collation（也就是有着最高得分的 collation）来同步分片**。

![Figure 6. Stateless client model](http://upload-images.jianshu.io/upload_images/127313-4a458b4eef3b5b6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为同步过程可能非常快，无状态客户端模型就可能在每个 collation 之间再混洗成。这不仅会减轻存储压力和开销，也会使系统更安全，因为频繁采样能够获得适应性攻击的抵抗能力。

Casper FFG 将会提供显式最终确定性阈值[after about 2.5 “epoch times”](http://ethresear.ch/t/casper-contract-and-full-pos/136/2)，也就是说，125 个区块时间[1][7]。如果在再混洗期间，验证人能够验证超过 `125 / PERIOD_LENGTH = 25` collation，分片系统能够从显式最终确定性中受益，确信从现在起的前 25 个 collation 可以最终确定。

当然了，如果在同步时有更多 collation 得到验证，就会更加安全。

## 结束语

希望我已经对当前以太坊的分片设计概念作出了简单介绍，以及显式最终确定性如何有益于分片机制。如果想要深入了解协议设计，请访问 [ETHResear.ch](https://ethresear.ch/) 和 [sharding doc](https://github.com/ethereum/sharding/blob/develop/docs/doc.md)。

如有任何错误，或是表述不清，欢迎指正！

特别感谢 Vitalik Buterin 的杰出工作，Jon Choi 督促我写下这篇文章，Dr.Chang-Wu Chen 修缮，Brian Chen 给予反馈 以及 Vlad Zamfir 的分享。

参考

1.  Jon Choi. Casper 101: [https://medium.com/@jonchoi/ethereum-casper-101-7a851a4f1eb0](https://medium.com/@jonchoi/ethereum-casper-101-7a851a4f1eb0)
2.  Vitalik Buterin. Sharding Document：
 [https://github.com/ethereum/sharding/blob/develop/docs/doc.md](https://github.com/ethereum/sharding/blob/develop/docs/doc.md)
3.  Vitalik Buterin and the contributors. Sharding FAQ：[https://github.com/ethereum/wiki/wiki/Sharding-FAQ](https://github.com/ethereum/wiki/wiki/Sharding-FAQ)
4.  Vitalik Buterin. Sharding Mindmap：[https://www.mindomo.com/mindmap/sharding-d7cf8b6dee714d01a77388cb5d9d2a01](https://www.mindomo.com/mindmap/sharding-d7cf8b6dee714d01a77388cb5d9d2a01)
5.  Vitalik Buterin. On Settlement Finality:[https://blog.ethereum.org/2016/05/09/on-settlement-finality/](https://blog.ethereum.org/2016/05/09/on-settlement-finality/)
6.  Ethresear.ch thread — The Stateless Client Concept:[https://ethresear.ch/t/the-stateless-client-concept/172](https://ethresear.ch/t/the-stateless-client-concept/172)
7.  Ethresear.ch thread — Casper contract and full POS:[https://ethresear.ch/t/casper-contract-and-full-pos/136/2](https://ethresear.ch/t/casper-contract-and-full-pos/136/2)


原文：https://medium.com/@icebearhww/ethereum-sharding-and-finality-65248951f649
