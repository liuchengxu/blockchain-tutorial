
- 无解的两将军问题
- 拜占庭将军问题
- 拜占庭容错

## 两将军问题

1975 年首次发表 http://hydra.infosys.tuwien.ac.at/teaching/courses/AdvancedDistributedSystems/download/1975_Akkoyunlu,%20Ekanadham,%20Huber_Some%20constraints%20and%20tradeoffs%20in%20the%20design%20of%20network%20communications.pdf  , 1978 年该问题正式命名为两将军问题(two generals problem)。说的是两个将军商量攻击一个共同的敌人，将军 1 为主，将军 2 为从。无论是将军 1 还是 2，仅凭其自身军队，谁也无法打败敌军。因此，他们必须合作，并在同一时间点同时发起进攻。这个问题看似很简单，却有一个避不开的问题：

将军 1 为了要告诉将军 2 进攻时间，必须要派信使穿过敌方军营通知将军 2。但是，信使可能会被敌方截获，因此信息也就无法送达将军 2。这就会导致将军 1 进攻的时候，将军 2 按兵不动，那么攻击就会失败。

即使将军 1 的信使将进攻时间顺利送达，将军 2 还必须要进行回应，告诉将军 1 确实收到了信息。这有点像三方的 [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)。那么又会出现上面的问题，信使可能会被敌军抓获。这就会造成无限的 ACK，两个将军永远也无法达成共识。

两将军问题已经被证明是[无解](https://en.wikipedia.org/wiki/Two_Generals%27_Problem#Proof)的。


## 拜占庭将军问题

[ Lamport, Shostak and Pease 1982](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.126.9525&rep=rep1&type=pdf)，这是两将军问题的广义版本。还是同一个场景，不同的地方是现在不只有两个将军，而是有更多的将军。此外还有更复杂的一点，这些将军里面可能有叛徒，叛徒会发出错误信心，比如说 9 点攻击，但是实际 9 点并不会攻击。


对于任意 m，如果有超过 3m 个将军，至多 m 个叛徒，算法 OM(m) 达成共识。

这表明只要有 2/3 的参与者是诚实的，那么算法就能达成共识。而一旦叛徒超过 1/3，就无法达成共识，无法阻止进攻，敌军自然胜利。


>参与者的目标是选择一个大部分都同意的同一个决定，而不是指定某一个决定。


7 个将军，两个叛徒，更多内容 [这里](http://marknelson.us/2007/07/23/byzantine/)

参考：

[1] https://medium.com/loom-network/understanding-blockchain-fundamentals-part-1-byzantine-fault-tolerance-245f46fe8419
