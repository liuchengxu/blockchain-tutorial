基本原型
========

## 区块链

有了区块，下面让我们来实现区块链。本质上，区块链只不过是一个有着特定结构的数据库，是一个有序，后向连接的列表。也就是说，区块按照插入的顺序进行存储，每个块都被连接到前一个块。这样的结构，能够让我们快速地获取链上的最新块，并且高效地通过哈希来检索一个块。

在 Golang 中，可以通过一个 array 和 map 来实现这个结构：array 存储有序的哈希（Golang 中 array 是有序的），map 存储 **hask -> block** 对(Golang 中, map 是无序的)。 但是在基本的原型阶段，我们只用到了 array，因为现在还不需要通过哈希来获取块。


```go
type Blockchain struct {
	blocks []*Block
}
```

这就是我们的第一个区块链！是不是出乎意料地简单?

现在，让我们能够给它添加一个区块：

```go
func (bc *Blockchain) AddBlock(data string) {
	prevBlock := bc.blocks[len(bc.blocks)-1]
	newBlock := NewBlock(data, prevBlock.Hash)
	bc.blocks = append(bc.blocks, newBlock)
}
```

完成！不过，就这样就完成了吗？

为了加入一个新的块，我们必须要有一个已有的块，但是，现在我们的链是空的，一个块都没有！所以，在任何一个区块链中，都必须至少有一个块。这样的块，也就是链中的第一个块，通常叫做创世块（**genesis block**）. 让我们实现一个方法来创建创世块：

```go
func NewGenesisBlock() *Block {
	return NewBlock("Genesis Block", []byte{})
}
```

现在，我们可以实现一个函数来创建有创世块的区块链：

```go
func NewBlockchain() *Blockchain {
	return &Blockchain{[]*Block{NewGenesisBlock()}}
}
```

检查一个我们的区块链是否如期工作：

```go
func main() {
	bc := NewBlockchain()

	bc.AddBlock("Send 1 BTC to Ivan")
	bc.AddBlock("Send 2 more BTC to Ivan")

	for _, block := range bc.blocks {
		fmt.Printf("Prev. hash: %x\n", block.PrevBlockHash)
		fmt.Printf("Data: %s\n", block.Data)
		fmt.Printf("Hash: %x\n", block.Hash)
		fmt.Println()
	}
}
```

输出：

```bash
Prev. hash:
Data: Genesis Block
Hash: aff955a50dc6cd2abfe81b8849eab15f99ed1dc333d38487024223b5fe0f1168

Prev. hash: aff955a50dc6cd2abfe81b8849eab15f99ed1dc333d38487024223b5fe0f1168
Data: Send 1 BTC to Ivan
Hash: d75ce22a840abb9b4e8fc3b60767c4ba3f46a0432d3ea15b71aef9fde6a314e1

Prev. hash: d75ce22a840abb9b4e8fc3b60767c4ba3f46a0432d3ea15b71aef9fde6a314e1
Data: Send 2 more BTC to Ivan
Hash: 561237522bb7fcfbccbc6fe0e98bbbde7427ffe01c6fb223f7562288ca2295d1
```

## 总结

我们创建了一个非常简单的区块链原型：它仅仅是一个数组构成的一系列区块，每个块都与前一个块相关联。真实的区块链要比这复杂得多。在我们的区块链中，加入新的块非常简单，也很快，但是在真实的区块链中，加入新的块需要很多工作：你必须要经过十分繁重的计算（这个机制叫做工作量证明），来获得添加一个新块的权力。并且，区块链是一个分布式数据库，并且没有单一决策者。因此，要加入一个新块，必须要被网络的其他参与者确认和同意（这个机制叫做共识（consensus））。还有一点，我们的区块链还没有任何的交易！

在接下来的文章的我们将会一一覆盖这些特性。

1. 原文源代码：[part_1](https://github.com/Jeiwan/blockchain_go/tree/part_1)

2. 区块哈希算法：[Block hashing algorithm](https://en.bitcoin.it/wiki/Block_hashing_algorithm)

原文链接：[Building Blockchain in Go. Part 1: Basic Prototype](https://jeiwan.cc/posts/building-blockchain-in-go-part-1/)

-----

进入 src 目录查看代码，执行 `make` 即可运行：

```bash
$ cd src
$ make
==> Go build
==> Running
Prev hash:
Data: Genesis Block
Hash: 4693b71eee96760de4b0f051083376dcbed2f0711a44294ee5fd42fbeacc9579

Prev hash: 4693b71eee96760de4b0f051083376dcbed2f0711a44294ee5fd42fbeacc9579
Data: Send 1 BTC to Ivan
Hash: 839380a2d0af1dc4686f16ade5423fecdc5f287db9322d9e18adcb4071e7c8ff

Prev hash: 839380a2d0af1dc4686f16ade5423fecdc5f287db9322d9e18adcb4071e7c8ff
Data: Send 2 more BTC to Ivan
Hash: b38052a029bd2b1b9d4bb478af45b4c88605e99bc64e49031ba06d21ad4b0b38
```
