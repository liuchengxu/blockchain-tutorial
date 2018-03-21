# DPOS 共识算法 - 缺失的白皮书

> 原文：https://steemit.com/dpos/@dantheman/dpos-consensus-algorithm-this-missing-white-paper
>
> 网络上已经有了好几个版本的译文，可能是原文写的没那么“平易近人”，这些译文我都看得不太懂:)
>

![Delegated Proof of Stake](https://steemitimages.com/DQmRR65U9ZYyMTVqtxRV6vu7AXj6GEouURjpEaoeSkHeHM8/image.png)

这篇“缺失的白皮书”是对委托权益证明(Delegated Proof of Stake, DPOS)的分析，旨在分析 DPOS 的工作原理及其鲁棒性（robust）的根源。[DPOS 的早期描述可以在 bitshares.org 找到](https://bitshares.org/technology/delegated-proof-of-stake-consensus/)；不过，那个描述里包含了很多与实际共识不大相关的内容。

本质上，所有区块链都是一种由交易驱动的确定性状态机。而共识，是就确定性交易顺序达成一致并过滤无效交易的过程。有各种不同的共识算法都可以产生等效的交易排序，但通过在多个区块链上长年累月的可靠运行，DPOS 已经证明其具备健壮性、安全性和有效性。

像所有共识算法一样，块生产者(俗话就是出块人)可能导致的最大损害是审查（censorship）。所有块的有效性必须基于确定性的开源状态机逻辑。

## DPOS 算法概要

DPOS 算法分为两部分：

1. 选择一组块生产者
2. 调度生产

选择出块人的过程，确保了利益相关方(stakeholder，通俗点也可以说是持币人)最终能有控制权，因为当网络不能顺利运行时，利益相关方的损失最大。就实际运行中如何达成共识而言，如何选择出块人其实几乎没有多大影响，因此，本文将重点介绍在选好出块人之后，如何达成共识。

为了帮助解释这个算法，我将假设 3 个块生产者 A，B 和 C。因为需要达成 2/3+1 的多数共识来解决所有情况，这个简化的模型将假设生产者 C 是打破僵局的那个人（也就是 2/3+1 里面的 1，胜负手）。在现实世界中，将有 21 个或更多的出块人。像工作量证明一样，一般规则是最长链胜出。任何时候，当一个诚实的节点看到一个有效的更长链，它都会从当前链切换到更长的这条链。

我将举例说明在大多数能够想到的网络条件下，DPOS 是如何运行的。这些例子应该可以帮助您理解为什么 DPOS 稳健且难以破坏。

## 正常操作(Normal Operation)

在正常操作下，出块人轮流每 3 秒钟出一个块。假设没有人错过自己的轮次，那么将会产生最长链。出块人在被调度轮次之外的任何时间段出块都是无效的。也就是说，如果没有轮到自己出块，出的任何块都是无效的。

![normal operation](https://steemitimages.com/DQmULkjtBAq4d3QypAZecgV4mAVJ9VDw9FjhbWJe7kWqZK8/image.png)

## 少数分叉(Minority Fork)

如果出现不超过节点总数三分之一的恶意或故障节点，那么可能会产生少数分叉（minority fork, 或者可以说是小群体分叉）。在这种情况下，少数分叉每 9 秒只能产生一个块，而多数分叉每 9 秒可以产生两个块。这样，诚实的 2/3 多数将永远比少数（的链）更长。

![minority fork](https://steemitimages.com/DQmaXit43FxdSQrn7PshKtJcnTY5SYJb1vSArf26ys34NDF/image.png)

## 离散少数的双重生产(Double Production by Disconnected Minority)

如果有很多“各自为政”的小群体，那么他们可以试图产生无数的分叉，但是因为少数人在出块速度上注定比多数人更慢，所以所有小群体的分叉都会比多数人的那条链短。

![double production by disconnected minority](https://steemitimages.com/0x0/https://steemitimages.com/DQmXsgRLQhmTVxjte48va1Qe6QePVzXUtmXRS8pjGPzUqGg/image.png)

## 网络碎片化(Network Fragmentation)

网络完全有可能碎片化，从而导致没有任何分叉拥有数量上占多数的出块人。在这种情况下，最长链将倒向最大的那个小群体（“矮子里面挑高个”）。当网络连通性恢复时，其他较小的小群体会自然切换到最长的那条链（也就是最大的小群体那条链），继而恢复明确的共识。

![network Fragmentation](https://steemitimages.com/DQmb4UaGGR8nNW5pmtoDE7Z9pvMd1LnKSpdi3utHUWeyfQZ/image.png)

有可能存在这样三个分叉，其中两个较长的分叉长度相同。在这种情况下，当第 3 个（较小）分叉的出块人重新加入网络时，就会打破平局。出块人总数为奇数，因此不可能长时间保持平局。稍后我们还会谈到出块人“混洗（shuffle）”，它使得出块顺序随机化，从而确保即使是出块人数目相同的两个分叉，也会以不同的速度增长，最终导致一个分叉胜出。

## 在线少数的双重生产(Double Production by Connected Minority)

在这种场景下，少数节点 B 在其时间段内产生了两个或更多的竞争块（alternative block）。下一个出块人（C）可以选择基于 B 产生的任意一个块继续往前走。如此一来，C 的这条链就成为最长链，而所有选择 B1 的节点都将切换到最长链。即使少数不良出块人试图广播再多的竞争块也无关紧要，因为它们作为最长链的一部分永远不会超过一轮(round，所有出块人挨个出块，转完一圈就是一轮)。

![double production by connected minority](https://steemitimages.com/0x0/https://steemitimages.com/DQmXstNcMtg5H6o1a955LZNGwbiCaygQnWzK81yeR8Uirvz/image.png)

## 最后不可逆块(Last Irreversible Block)

在网络碎片化的情况下，在相当长的一段时间内，很有可能有多个分叉都持续不断地增长。长远来看，最长的链终将获胜，但观察者（observer）需要一种确切的手段，以此来确定一个块是否在增长最快的链上。这可以通过看是否有 2/3+1 多数出块人的确认来确定。

在下图中，块 B 已被 C 和 A 所确认，即表示有 2/3+1 多数确认，由此我们可以推断没有其它链会比这条链更长 – 如果 2/3 的出块人是诚实的。

![last inreversible block](https://steemitimages.com/DQmWjbpfju5vj1EGBtfLKSGcM4CpHmYppgB9cfev2dFLtyF/image.png)

请注意，这个“规则”类似于比特币的 6 个块确认。一些“聪明”人也许可以谋划一系列事件，使得两个节点的最后不可逆块不同。这种极端的例子要求攻击者能完全控制通信延迟，并且在几分钟内控制两次--而不仅仅是一次。即便这真的发生了，最长链胜出的长期规则仍然适用。我们认为这种攻击的可能性足够接近 0，且经济后果无关紧要，因此不足为虑。

## 出块人不足(Lack of Quorum of Producers)

有一种不太可能会出现的情况，即没有明确到底有多少个出块人，在这种情况下，少数人还是可以继续出块。在这些块里，利益相关方可以包含更改投票的交易。这些投票可以选出一组新的出块人，并将出块参与率恢复到 100％。一旦如此，这条小群体的链最终将超过所有其他出块参与率低于 100％ 的链。

在此过程中，所有观察者都会知道，在出现一条参与率超过 67% 的链之前，网络都处于不稳定的状态。如果有人选择在此条件下进行交易，那么他将承受与接受不到 6 个确认类似的风险。他们也知道存在这样的小概率事件，即最终的共识（或者说最长链）出现在另一个不同的分叉上。在实际操作中，这种情况仍然要比接受少于 3 个比特币确认要安全的多。

## 多数出块人出现问题(Corruption of Majority of Producers)

如果多数出块人出现问题，那么他们可能产生无限的分叉，每个分叉都看起来以 2/3 多数确认向前推进。这种情况下，最后不可逆块算法蜕变为最长链算法。最长链就是为最多人认同的链，而这条链实际将由少数剩下的诚实节点决定。这种行为不会持续很长时间，因为利益相关方最终会投票替换掉这些出块人。

![corruption of majority of producers](https://steemitimages.com/0x0/https://steemitimages.com/DQmZvNhkNbLLmvFQZEDhiXrCkmdEHm4q5tmL5Jo7ngJFWJG/image.png)

## 交易权益证明（Transactions as Proof of Stake, TaPoS）

用户为一个交易签名，是基于对区块链状态的一个假设，而这个假设又基于他们对最近几个块的“认识”（perception）。如果最长链的共识发生改变，则很可能会使签名者之前签名时所依据的假设失效。

就 TaPoS 而言，所有交易都包含最近一个块的哈希，如果该块在区块链中不存在，那么认为这些交易是无效的。任何在孤儿分叉(orphaned fork)上给交易签名的人，最终都会发现该交易无效，且无法迁移到主分叉。

这个过程有个附带的好处，它可以抵御试图产生替代链的长程攻击（long-range attack）。每次交易时，利益相关方都直接对区块链做出确认。随着时间的推移，所有的块都是由所有利益相关方确认过的，这在一条伪造的链（forged chain）所做不到的。

## 确定性出块人混洗(Deterministic Producer Shuffling)

在上面我们所展示的所有案例中，出块人按循环调度出块。实际上，每出 N 个块（N 是出块人数量），出块人集合都会进行一次混洗。这种随机性确保了出块人 B 不会总是忽略出块人 A，并且当出现多个数量出块人相同的分叉时，最终会有一个分叉胜出。

## 结论

在每一个想得到的自然网络破坏下，DPOS 都是健壮的，甚至在面对大部分出块人作弊时，也是安全的。不像其它共识算法，当大多数出块人出现问题时，DPOS 仍然可以继续工作。在此过程中，社区可以投票替换掉不合格的出块人，直到恢复 100％ 参与率。在如此高强度和变化多端的故障条件下，我还不知道有任何其它算法也可以依然保持如此健壮。

DPOS 之所以能有这么高的安全性，归根结底来自于其选择出块人和验证节点质量的算法。通过赞成投票制（approval voting），可以确保即使一个人拥有 50％ 的有效投票权，也不能独自选择一个出块人。DPOS 的设计初衷是在良好的网络连接，诚实节点 100％ 参与共识的情况下优化性能，这使得 DPOS 有能力在平均只有 1.5 秒的时间内以 99.9％ 的确定性确认交易，同时能够以一种优雅和可检测的方式降级 – 降级也不是什么大事。

其他共识算法的设计初衷是，节点不诚实且网络条件恶劣。这种设计的最终结果就是，网络的性能更差，延迟更高，更新开销大，并且在 33% 节点出现问题的情况下就会彻底停止正常运转。

在 BitShares 成功运行三年，Steem 运行一年期间，我们经历了各种各样的网络条件和软件错误。对于各种状况，DPOS 都成功地得以应对，并且证明了在维护共识的同时，处理了比任何其它区块链更多的交易。
