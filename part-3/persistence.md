持久化和命令行接口
==================

## 持久化

让我们从 `NewBlockchain` 函数开始。在之前的实现中，`NewBlockchain` 会创建一个新的 `Blockchain` 实例，并向其中加入创世块。而现在，我们希望它做的事情有：

1. 打开一个数据库文件
2. 检查文件里面是否已经存储了一个区块链
3. 如果已经存储了一个区块链：
    1. 创建一个新的 `Blockchain` 实例
    2. 设置 `Blockchain` 实例的 tip 为数据库中存储的最后一个块的哈希
4. 如果没有区块链：
    1. 创建创世块
    2. 存储到数据库
    3. 将创世块哈希保存为最后一个块的哈希
    4. 创建一个新的 `Blockchain` 实例，初始时 tip 指向创世块（tip 有尾部，尖端的意思，在这里 tip 存储的是最后一个块的哈希）

代码大概是这样：

```go
func NewBlockchain() *Blockchain {
	var tip []byte
	db, err := bolt.Open(dbFile, 0600, nil)

	err = db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))

		if b == nil {
			genesis := NewGenesisBlock()
			b, err := tx.CreateBucket([]byte(blocksBucket))
			err = b.Put(genesis.Hash, genesis.Serialize())
			err = b.Put([]byte("l"), genesis.Hash)
			tip = genesis.Hash
		} else {
			tip = b.Get([]byte("l"))
		}

		return nil
	})

	bc := Blockchain{tip, db}

	return &bc
}
```

来一段一段地看下代码：

```go
db, err := bolt.Open(dbFile, 0600, nil)
```

这是打开一个 BoltDB 文件的标准做法。注意，即使不存在这样的文件，它也不会返回错误。

```go
err = db.Update(func(tx *bolt.Tx) error {
...
})
```

在 BoltDB 中，数据库操作通过一个事务（transaction）进行操作。有两种类型的事务：只读（read-only）和读写（read-write）。这里，打开的是一个读写事务（`db.Update(...)`），因为我们可能会向数据库中添加创世块。

```go
b := tx.Bucket([]byte(blocksBucket))

if b == nil {
	genesis := NewGenesisBlock()
	b, err := tx.CreateBucket([]byte(blocksBucket))
	err = b.Put(genesis.Hash, genesis.Serialize())
	err = b.Put([]byte("l"), genesis.Hash)
	tip = genesis.Hash
} else {
	tip = b.Get([]byte("l"))
}
```

这里是函数的核心。在这里，我们先获取了存储区块的 bucket：如果存在，就从中读取 `l` 键；如果不存在，就生成创世块，创建 bucket，并将区块保存到里面，然后更新 `l` 键以存储链中最后一个块的哈希。

另外，注意创建 `Blockchain` 一个新的方式：

```go
bc := Blockchain{tip, db}
```

这次，我们不在里面存储所有的区块了，而是仅存储区块链的 `tip`。另外，我们存储了一个数据库连接。因为我们想要一旦打开它的话，就让它一直运行，直到程序运行结束。因此，`Blockchain` 的结构现在看起来是这样：

```go
type Blockchain struct {
	tip []byte
	db  *bolt.DB
}
```

接下来我们想要更新的是 `AddBlock` 方法：现在向链中加入区块，就不是像之前向一个数组中加入一个元素那么简单了。从现在开始，我们会将区块存储在数据库里面：

```go
func (bc *Blockchain) AddBlock(data string) {
	var lastHash []byte

	err := bc.db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		lastHash = b.Get([]byte("l"))

		return nil
	})

	newBlock := NewBlock(data, lastHash)

	err = bc.db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		err := b.Put(newBlock.Hash, newBlock.Serialize())
		err = b.Put([]byte("l"), newBlock.Hash)
		bc.tip = newBlock.Hash

		return nil
	})
}
```

继续来一段一段分解开来：

```go
err := bc.db.View(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte(blocksBucket))
	lastHash = b.Get([]byte("l"))

	return nil
})
```

这是 BoltDB 事务的另一个类型（只读）。在这里，我们会从数据库中获取最后一个块的哈希，然后用它来挖出一个新的块的哈希：

```go
newBlock := NewBlock(data, lastHash)
b := tx.Bucket([]byte(blocksBucket))
err := b.Put(newBlock.Hash, newBlock.Serialize())
err = b.Put([]byte("l"), newBlock.Hash)
bc.tip = newBlock.Hash
```

## 检查区块链

现在，产生的所有块都会被保存到一个数据库里面，所以我们可以重新打开一个链，然后向里面加入新块。但是在实现这一点后，我们失去了之前一个非常好的特性：再也无法打印区块链的区块了，因为现在不是将区块存储在一个数组，而是放到了数据库里面。让我们来解决这个问题！

BoltDB 允许对一个 bucket 里面的所有 key 进行迭代，但是所有的 key 都以字节序进行存储，而且我们想要以区块能够进入区块链中的顺序进行打印。此外，因为我们不想将所有的块都加载到内存中（因为我们的区块链数据库可能很大！或者现在可以假装它可能很大），我们将会一个一个地读取它们。故而，我们需要一个区块链迭代器（`BlockchainIterator`）：

```go
type BlockchainIterator struct {
	currentHash []byte
	db          *bolt.DB
}
```

每当要对链中的块进行迭代时，我们就会创建一个迭代器，里面存储了当前迭代的块哈希（`currentHash`）和数据库的连接（`db`）。通过 `db`，迭代器逻辑上被附属到一个区块链上（这里的区块链指的是存储了一个数据库连接的 `Blockchain` 实例），并且通过 `Blockchain` 方法进行创建：

```go
func (bc *Blockchain) Iterator() *BlockchainIterator {
	bci := &BlockchainIterator{bc.tip, bc.db}

	return bci
}
```

注意，迭代器的初始状态为链中的 tip，因此区块将从尾到头（创世块为头），也就是从最新的到最旧的进行获取。实际上，**选择一个 tip 就是意味着给一条链“投票”**。一条链可能有多个分支，最长的那条链会被认为是主分支。在获得一个 tip （可以是链中的任意一个块）之后，我们就可以重新构造整条链，找到它的长度和需要构建它的工作。这同样也意味着，一个 tip 也就是区块链的一种标识符。

`BlockchainIterator` 只会做一件事情：返回链中的下一个块。

```go
func (i *BlockchainIterator) Next() *Block {
	var block *Block

	err := i.db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		encodedBlock := b.Get(i.currentHash)
		block = DeserializeBlock(encodedBlock)

		return nil
	})

	i.currentHash = block.PrevBlockHash

	return block
}
```

这就是数据库部分的内容了！

----

- 上一节: [持久化和命令行接口](persistence-and-cli.md)
- 下一节: [命令行接口](cli.md)
