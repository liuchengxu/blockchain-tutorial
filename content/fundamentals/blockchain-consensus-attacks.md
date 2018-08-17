# 区块链共识级别的攻击

## Double spending attacks

推翻已经确认的交易。

- persistence

## Grinding attacks

恶意节点试图影响 slot leader selection 的过程，加大自己被选中为 slot leader 的概率。

这种攻击主要是由于随机性来源于链本身的原始数据，比如块头和交易，恶意节点可以尝试多个可能成为 slot leader 的 block header。

## Transaction denial attacks

阻止一笔交易被确认。恶意节点监控某个特殊的账户，不让它发交易。

liveness

## Desynchronization attacks

诚实节点无法与网络同步。

## Eclipse attacks

## 51% attacks

## Bribery attacks

- Short Range attack

Long-Range attack + checkpoint = short range attack

## Long-range attacks

所谓长程攻击，指的是恶意节点试图从创世块开始对链进行分叉，分叉链可能或包含于主链不同的交易和块，因此也叫 **Alternative History** 或者 **History Revision** 攻击。Long Range, Alternative History, Alternate History, History Revision 在很多情况下是通用的概念。

长程攻击之所以存在的主要原因是 Weak Subjectivity。

### Weak Subjectivity

这个问题主要会影响刚刚加入到网络的新节点和长时间离线后重新加入的节点。

当一个新节点刚刚加入网络时，创世块肯定是有的，也是毫无争议的。但是除了创世块，节点还会看到目前有好几条链 -- 主链与分叉链，即使并非恶意分叉，PoW 在末端自身也极易分叉。然后，节点并不能从这些链中判断出哪个是主链。

长时间离线后重新加入的节点也是如此。PoS 也是最长链原则，拥有最多块数的叉将成为主链。在 PoW 中，要投入大量算力才能获取出块权，对于 PoW 来说，最长链十分有效。

在 PoW 中，除非遭受 51% 攻击，否则不可能从创世块开始分叉。要伪造跟主链同样长度的分叉，需要投入大量的算力。因此，最长链原则已经可以解决 Weak Subjectivity 问题。

### Costless Simulation

对于 PoS 来说，最长链就不够用了，因为这里又出现了一个叫 Costless Simulation 的问题。PoS 的验证人不需要像矿工一样必须要进行大量计算，验证人只需要从交易池中取出交易，打包入块，最后广播出去。因此伪造跟主链一样长的链，毫无成本可言。

Weak Subjectivity 加上 Costless Simulation 导致最长链原则对 PoS 不再适用，长程攻击由此产生。

### Long Range Attacks

1. simple
2. posterior corruption ==> Key Evolving Signature Scheme
3. stake bleeding

- https://blog.positive.com/rewriting-history-a-brief-introduction-to-long-range-attacks-54e473acdba9
- [Stake-Bleeding Attacks on Proof-of-Stake Blockchains](https://eprint.iacr.org/2018/248.pdf)

## Nothing at stake attacks

当发生分叉时，验证人的最优策略是承认每一条分叉，并在所有分叉上继续挖。这样无论哪一条叉最终胜出，验证人都能获得奖励。

- https://en.wikipedia.org/wiki/Proof-of-stake
- https://blog.goldmint.io/nothing-at-stake-and-longrange-attack-in-pos-4ec486f1fc89
- https://github.com/ethereum/wiki/wiki/Problems
- https://pivx.org/nothing-considered-a-look-at-nothing-at-stake-vulnerability-for-cryptocurrencies/
- https://medium.com/@abhisharm/understanding-proof-of-stake-through-its-flaws-part-2-nothing-s-at-stake-8d12d826956c
- https://medium.com/coinmonks/understanding-proof-of-stake-the-nothing-at-stake-theory-1f0d71bc027

## Past majority attacks

nothing at stake 的一种特殊情况。

## Selfish-mining

自私挖矿，恶意挖到块以后，并不广播出去，而是偷偷藏在手里，然后继续挖。当监听到有节点挖出同样高度的块时，迅速将手里的块广播出去，如果恶意节点的网络更好, 那么恶意节点手中同高度的块就会被接受，如此一来，恶意节点就会更有优势挖到下面的块。

## Reference

- Ouroboros
