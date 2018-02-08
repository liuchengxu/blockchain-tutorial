卡尔达诺入门必备
================

本文为卡尔达诺（Cardano）的入门读物，主要内容为卡尔达诺的整个项目概览：

- 卡尔达诺是什么，为什么，有何独特之处，以及团队组织。
- ADA 是什么，乌洛波罗斯（Ouroboros）共识。

## 卡尔达诺

### 是什么，为什么

卡尔达诺是一个去中心化的公有区块链和加密货币项目，它目前正在开发一个智能合约平台，该平台旨在提供更多超越以往任何协议的高级功能。它是第一个衍生自科学哲学的区块链平台，主要以研究驱动，开发团队网罗了全球顶尖的工程师和研究人员。

通俗来讲，卡尔达诺以**可证的权益证明共识（provable PoS）**乌洛波罗斯（Ouroboros）为中心，集比特币（Bitcoin），以太坊（Ethereum，智能合约），波卡（Polkadot，跨链）之大成。卡尔达诺的智能合约将使用第三代基于 LLVM 的虚拟机 [IELE](https://iohk.io/blog/iele-a-new-virtual-machine-for-the-blockchain)，通过侧链来实现跨链功能。

卡尔达诺的最初目标是希望可以改善当前加密货币的设计与开发模式，最终愿景是希望可以提供一个更加平衡，且可持续发展的生态系统，并满足用户寻求其他系统整合的需求。

### 有何独特之处

卡尔达诺是目前第一个通过学术界同行评审的区块链项目，可以说在行业中是只此一家，绝无仅有。另一独特之处是，Cardano 是由 Haskell 语言实现，Haskell 被认为最安全的编程语言之一，它可以将错误的发生机率降至最低，同时为平台的安全性提供保证。当然了，也有人认为使用 Haskell 反而是个劣势，因为作为一个小众语言，这让开发门槛高了不少，可能会有因此让不少 ADA 开发爱好者 “报国无门”。

### 团队组织

先来说一下相关组织。卡尔达诺的背后主要有三个组织。第一个是创立于瑞士的卡尔达诺基金会，它是一个非营利组织。卡尔达诺基金会的核心使命是培育、促进发展与教育卡尔达诺用户和商业社区，接洽监管与当局商业事务。卡尔达诺背后的第二个实体组织是 IOHK，这是一家加密货币研发的先驱公司，该公司开发卡尔达诺平台的合约一直持续到 2020 年。卡尔达诺项目的第三个业务合作伙伴是 Emurgo，该公司为创业投资咨询公司, 同时协助企业建立卡尔达诺区块链系统。

再来谈一下团队，用个词来说，就是高端大气上档次。在这里仅简单介绍两个人，更多内容可自行点击 [team](https://iohk.io/team/) 查看。

[Aggelos Kiayias](https://iohk.io/team/aggelos-kiayias/) 是爱丁堡大学主管网络安全和隐私的教授，在相关学术期刊和会议上发表过超过 100 篇论文，[这里](https://scholar.google.ae/citations?user=P_L_vZAAAAAJ&hl=zh-CN&oi=ao) 是他的 Google scholar 主页，有兴趣的可以去了解一下。他是 ouroboros 白皮书的主要作者，也是 IOHK 的首席科学家。

[Philip Wadler](https://iohk.io/team/philip-wadler/) 是爱丁堡大学理论计算机科学系的教授，他参与贡献了 Haskell，Java 和 XQuery 语言的设计，也是<<Introduction to Functional Programming (Prentice Hall, 1988)>>，<< XQuery from the Experts (Addison Wesley, 2004)>> 和 <<Generics and Collections in Java (O’Reilly, 2006)>> 等著作的合著者之一。[这里](https://scholar.google.ae/citations?user=Iz-3VFQAAAAJ&hl=zh-CN&oi=ao) 是他的 Google scholar 主页，引用数超过 20000，在学术界也是泰斗无疑了。

## 什么是 ADA

每个公有链都有一个代币（token），ADA 就是是卡尔达诺区块链上的代币。ADA 是卡尔代币区块链平台上的基础代币，如果有侧链，侧链也会有自己的代币。

## 什么是乌洛波罗斯（Ouroboros）

卡尔达诺采用一种革命性的新权益证明（PoS）算法，称为乌洛波罗斯（ouroboros），它决定了各个节点如何达成网络一致性。该算法是整个基础架构的关键所在，是区块链技术的重大创新。

目前大多区块链采用的是 PoW 共识，但是该共识有不少缺点，比如资源浪费。在工作量证明（PoW）中，矿工投入运算能力来竞争下一个块的出块权。PoW 的关键在于解决了一个随机的“领导人选择（leader election）”问题，也就是选出一个矿工来出下一个块。

在权益证明中，依照区块链账本中股权者所拥有权益的比例，随机选取选择下ㄧ个出块人。为了确保区块链的安全性，选择股权者来产生区块的方法必须是真随机的。为了实现领导者选举（leader election）过程的随机性，乌洛波罗斯的创新是通过安全、多方执行掷硬币协议来达成这点。

实际上，权益证明的概念由来已久，也有不少项目已经采用了该算法，但是已有的 PoS 有着诸多缺陷，并且无法证明其安全性。乌洛波洛斯是第一个安全性经过形式化证明的权益证明共识，它由 IOHK 首席科学家 [Aggelos Kiayias](https://iohk.io/team/aggelos-kiayias/) 教授领导的团队设计而成，并且通过了学术界同行的一致评审，论文在 [这里](https://eprint.iacr.org/2016/889.pdf)，更多文章也在下面的学术论文链接中找到。

## 其他

由于卡尔达诺的相关资源实在是过于分散，我们在这里也做了简单的总结：

官方网站：

- [cardanohub](https://www.cardanohub.org/en/home)
- [input | output](https://iohk.io/)
- [emurgo](http://emurgo.io/)
- [cardanofoundation](https://cardanofoundation.org/)

Twitter:

1. [Input Output](https://twitter.com/InputOutputHK)
2. [Cardano Foundation](https://twitter.com/CardanoStiftung)
3. [Cardano Community](https://twitter.com/cardanocom)
4. [ADA](https://twitter.com/ADAcoin_)
5. [emurgo](https://twitter.com/emurgo_io)

论文文档：

- [学术论文](https://www.cardanohub.org/en/academic-papers/)
- [卡尔达诺清算层文档](https://cardanodocs.com/introduction/)

GitHub：

- [cardano-sl](https://github.com/input-output-hk/cardano-sl)
- [input out community](https://github.com/input-output-hk)

团队：

- [iohk team](https://iohk.io/team/)

参考资料：

[1] https://www.cardanohub.org/
