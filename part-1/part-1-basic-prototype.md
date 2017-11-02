Part 1: 基本原型
================

## 目录

<!-- vim-markdown-toc GFM -->

* [引言](#引言)
* [区块](#区块)
* [区块链](#区块链)
* [总结](#总结)

<!-- vim-markdown-toc -->

## 引言

区块链是 21 世纪最具革命性的技术之一，它仍然处于不断成长的阶段，而且还有很多潜力尚未显现出来。 本质上，区块链只是一个分布式数据库而已。 不过，使它独一无二的是，区块链是一个公开的数据库，而不是一个私人数据库，也就是说，每个使用它的人都有一个完整或部分的副本。 只有经过其他“数据库管理员”的同意，才能向数据库中添加新的记录。 此外，也正是由于区块链，才使得加密货币和智能合约成为现实。

在本系列文章中，我们将实现一个简化版的区块链，并基于它来构建一个简化版的加密货币。

## 区块

让我们从 “区块链” 中的 “区块” 谈起。在区块链中，存储有效信息的是区块（block）。比如，比特币区块存储的有效信息，就是比特币交易（transaction），交易信息也是所有加密货币的本质。除此以外，区块还包含了一些技术信息，比如版本，当前时间戳和前一个区块的哈希。

 在本文，我们并不会实现一个像比特币技术规范所描述那样的区块链，而是实现一个简化版的区块链，它仅包含了一些关键信息。看起来就像是这样：

```go
type Block struct {
	Timestamp     int64
	Data          []byte
	PrevBlockHash []byte
	Hash          []byte
}
```

- `Timestamp`: 当前时间戳，也就是区块创建的时间。
- `Data`: 区块存储的实际有效信息。
- `PrevBlockHash`: 前一个块的哈希。
- `Hash`: 当前块的哈希。

在比特币技术规范中，`Timestamp`，`PrevBlockHash`, `Hash` 是区块头（block header），区块头是一个单独的数据结构。而交易，也就是这里的 `Data`, 是另一个单独的数据结构。为了简便起见，目前把这两个混合在了一起。

那么，要如何计算哈希呢？计算哈希，是区块链一个非常重要的部分。正是由于这个特性，才使得区块链是安全的。计算一个哈希，是在计算上非常困难的一个操作。即使在高速电脑上，也要花费不少时间 (这就是为什么人们会购买 GPU 来挖比特币) 。这是一个有意为之的架构设计，它故意使得加入新的区块十分困难，继而保证区块一旦被加入以后，就很难再进行修改。在本系列未来几篇文章中，我们将会讨论和实现这个机制。

目前，我们仅取了 Block 结构的部分字段（`Timestamp`, `Data` 和 `PrevBlockHash`），并将它们相互连接起来，然后在连接后的结果上计算一个 SHA-256 的哈希. 让我们在 `SetHash` 方法完成这个任务：

```go
func (b *Block) SetHash() {
	timestamp := []byte(strconv.FormatInt(b.Timestamp, 10))
	headers := bytes.Join([][]byte{b.PrevBlockHash, b.Data, timestamp}, []byte{})
	hash := sha256.Sum256(headers)

	b.Hash = hash[:]
}
```

接下来，按照 Golang 的惯例，我们会实现一个用于简化创建区块的函数 `NewBlock`：

```go
func NewBlock(data string, prevBlockHash []byte) *Block {
	block := &Block{time.Now().Unix(), []byte(data), prevBlockHash, []byte{}}
	block.SetHash()
	return block
}
```

这就是区块部分的全部内容了！

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
