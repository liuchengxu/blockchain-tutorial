基本原型
========

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


