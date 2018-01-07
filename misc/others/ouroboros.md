self mining

1. 具有可证的安全性
2. 采用 Pos 而不是 PoW，并提出了一个新的奖励机制，在该机制下，良好的行为处于一个近似纳什均衡的状态，并且可以缓解自私挖矿等。

PoW 的一大弊端是消耗资源。

PoW 的关键作用在于解决了一种随机的“领导人选举（leader election）”问题，也就是选择一个矿工来出下一个块。

出块机会跟算力成正相关，而“自私挖矿”可能会破坏这种机制。


于是引入 PoS，与投入计算资源不同，PoS 投入的是 *stake*，通常也就是所拥有的代币（token），有多少权益，就有多大的机会出块。

之前的 PoS 的区块链协议，只是从实践中证明了可以抵抗某些类型的攻击，但是并没有严格的形式化证明。所谓启发式的（heuristic）PoS 已经早已存在，比如 NXT, Neucoin, Blackcoin, Tendermint, Bitshares. 这些协议都是有不足的。

Proof-of-Work Proof-of-space proof-of-space-time

PoS 的基本问题是模拟领导人选举过程。引入 entropy 熵

“grinding” 漏洞


提出 ouroboros 可证的 PoS 系统

1. 一个模型，形式化 PoS 区块链协议。关注 persistence and liveness

persistence：如果一个节点认为一笔交易稳定，那么其他正常节点也认为如此。该属性的确定性由一个安全参数 k 确定，比如 超过 k 个块高度。

liveness: 一笔合法交易存在一定时间后，比如 u 时间，就会认为是 stable。

2. 提出一个新的区块链协议。


使得 PoS 独特之处在于，PoS 的权益随着时间变化，信任假设也会随着时间演进。

基本问题是 leader election
PoW   - randomized leader election  -- invest computational resources

PoS - *stake*  proportionally  -- coin


stakeholder  同时赋予义务与权力 assign work as well as rewards


inspiredy by the two properties, persistence and liveness


First, provice a formalized model

Second, provide a blockchain protocol. coin-flipping protocol produces the randomness of the leader election process.

Third, guarantee the persistence and liveness.

Forth, a novel incentive strcture of the protocol. Nash equilibrium.

Fifth, a stake delegation  algorithm.
