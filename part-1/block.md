基本原型
========

## 区块

首先从 “区块” 谈起。在区块链中，真正存储有效信息的是区块（block）。而在比特币中，真正有价值的信息就是交易（transaction）。实际上，交易信息是所有加密货币的价值所在。除此以外，区块还包含了一些技术实现的相关信息，比如版本，当前时间戳和前一个区块的哈希。

不过，我们要实现的是一个简化版的区块链，而不是一个像比特币技术规范所描述那样成熟完备的区块链。所以在我们目前的实现中，区块仅包含了部分关键信息，它的数据结构如下：

```go
type Block struct {
	Timestamp     int64
	Data          []byte
	PrevBlockHash []byte
	Hash          []byte
}
```

字段            | 解释
:----:          | :----
`Timestamp`     | 当前时间戳，也就是区块创建的时间
`PrevBlockHash` | 前一个块的哈希，即父哈希
`Hash`          | 当前块的哈希
`Data`          | 区块存储的实际有效信息，也就是交易

我们这里的 `Timestamp`，`PrevBlockHash`, `Hash`，在比特币技术规范中属于区块头（block header），区块头是一个单独的数据结构。
完整的 [比特币的区块头（block header）结构](https://en.bitcoin.it/wiki/Block_hashing_algorithm) 如下：

Field          | Purpose                                                    | Updated when...                                         | Size (Bytes)
:----          | :----                                                      | :----                                                   | :----
Version        | Block version number                                       | You upgrade the software and it specifies a new version | 4
hashPrevBlock  | 256-bit hash of the previous block header                  | A new block comes in                                    | 32
hashMerkleRoot | 256-bit hash based on all of the transactions in the block | A transaction is accepted                               | 32
Time           | Current timestamp as seconds since 1970-01-01T00:00 UTC    | Every few seconds                                       | 4
Bits           | Current target in compact format                           | The difficulty is adjusted                              | 4
Nonce          | 32-bit number (starts at 0)                                | A hash is tried (increments)                            | 4

下面是比特币的 golang 实现 btcd 的 [BlockHeader](https://github.com/btcsuite/btcd/blob/01f26a142be8a55b06db04da906163cd9c31be2b/wire/blockheader.go#L20-L41) 定义:

```go
// BlockHeader defines information about a block and is used in the bitcoin
// block (MsgBlock) and headers (MsgHeaders) messages.
type BlockHeader struct {
    // Version of the block.  This is not the same as the protocol version.
    Version int32

    // Hash of the previous block in the block chain.
    PrevBlock chainhash.Hash

    // Merkle tree reference to hash of all transactions for the block.
    MerkleRoot chainhash.Hash

    // Time the block was created.  This is, unfortunately, encoded as a
    // uint32 on the wire and therefore is limited to 2106.
    Timestamp time.Time

    // Difficulty target for the block.
    Bits uint32

    // Nonce used to generate the block.
    Nonce uint32
}
```

而我们的 `Data`, 在比特币中对应的是交易，是另一个单独的数据结构。为了简便起见，目前将这两个数据结构放在了一起。在真正的比特币中，[区块](https://en.bitcoin.it/wiki/Block#Block_structure) 的数据结构如下：

Field               | Description                                  | Size
:----               | :----                                        | :----
Magic no            | value always 0xD9B4BEF9                      | 4 bytes
Blocksize           | number of bytes following up to end of block | 4 bytes
Blockheader         | consists of 6 items                          | 80 bytes
Transaction counter | positive integer VI = VarInt                 | 1 - 9 bytes
transactions        | the (non empty) list of transactions         | <Transaction counter>-many transactions

在我们的简化版区块中，还有一个 `Hash` 字段，那么，要如何计算哈希呢？哈希计算，是区块链一个非常重要的部分。正是由于它，才保证了区块链的安全。计算一个哈希，是在计算上非常困难的一个操作。即使在高速电脑上，也要耗费很多时间 (这就是为什么人们会购买 GPU，FPGA，ASIC 来挖比特币) 。这是一个架构上有意为之的设计，它故意使得加入新的区块十分困难，继而保证区块一旦被加入以后，就很难再进行修改。在接下来的内容中，我们将会讨论和实现这个机制。

目前，我们仅取了 `Block` 结构的部分字段（`Timestamp`, `Data` 和 `PrevBlockHash`），并将它们相互拼接起来，然后在拼接后的结果上计算一个 SHA-256，然后就得到了哈希.

```
Hash = SHA256(PrevBlockHash + Timestamp + Data)
```

在 `SetHash` 方法中完成这些操作：

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

这就是区块的全部内容了！在这里，我们需要了解区块的数据结构，如何计算 `Hash`。

----

- 上一节: [基本原型](basic-prototype.md)
- 下一节: [区块链](blockchain.md)
