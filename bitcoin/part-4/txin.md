交易（1）
========

## 交易输入

这里是输入：

```go
type TXInput struct {
	Txid      []byte
	Vout      int
	ScriptSig string
}
```

正如之前所提到的，一个输入引用了之前交易的一个输出：`Txid` 存储的是之前交易的 ID，`Vout` 存储的是该输出在那笔交易中所有输出的索引（因为一笔交易可能有多个输出，需要有信息指明是具体的哪一个）。`ScriptSig` 是一个脚本，提供了可解锁输出结构里面 `ScriptPubKey` 字段的数据。如果 `ScriptSig` 提供的数据是正确的，那么输出就会被解锁，然后被解锁的值就可以被用于产生新的输出；如果数据不正确，输出就无法被引用在输入中，或者说，无法使用这个输出。这种机制，保证了用户无法花费属于其他人的币。

再次强调，由于我们还没有实现地址，所以目前 `ScriptSig` 将仅仅存储一个用户自定义的任意钱包地址。我们会在下一篇文章中实现公钥（public key）和签名（signature）。

来简要总结一下。输出，就是 “币” 存储的地方。每个输出都会带有一个解锁脚本，这个脚本定义了解锁该输出的逻辑。每笔新的交易，必须至少有一个输入和输出。一个输入引用了之前一笔交易的输出，并提供了解锁数据（也就是 `ScriptSig` 字段），该数据会被用在输出的解锁脚本中解锁输出，解锁完成后即可使用它的值去产生新的输出。

每一笔输入都是之前一笔交易的输出，那么假设从某一笔交易开始不断往前追溯，它所涉及的输入和输出到底是谁先存在呢？换个说法，这是个鸡和蛋谁先谁后的问题，是先有蛋还是先有鸡呢？

## 先有蛋

在比特币中，是先有蛋，然后才有鸡。输入引用输出的逻辑，是经典的“蛋还是鸡”问题：输入先产生输出，然后输出使得输入成为可能。在比特币中，最先有输出，然后才有输入。换而言之，第一笔交易只有输出，没有输入。

当矿工挖出一个新的块时，它会向新的块中添加一个 **coinbase** 交易。coinbase 交易是一种特殊的交易，它不需要引用之前一笔交易的输出。它“凭空”产生了币（也就是产生了新币），这是矿工获得挖出新块的奖励，也可以理解为“发行新币”。

在区块链的最初，也就是第一个块，叫做创世块。正是这个创世块，产生了区块链最开始的输出。对于创世块，不需要引用之前的交易输出。因为在创世块之前根本不存在交易，也就没有不存在交易输出。

来创建一个 coinbase 交易：

```go
func NewCoinbaseTX(to, data string) *Transaction {
	if data == "" {
		data = fmt.Sprintf("Reward to '%s'", to)
	}

	txin := TXInput{[]byte{}, -1, data}
	txout := TXOutput{subsidy, to}
	tx := Transaction{nil, []TXInput{txin}, []TXOutput{txout}}
	tx.SetID()

	return &tx
}
```

coinbase 交易只有一个输出，没有输入。在我们的实现中，它表现为 `Txid` 为空，`Vout` 等于 -1。并且，在当前实现中，coinbase 交易也没有在 `ScriptSig` 中存储脚本，而只是存储了一个任意的字符串 `data`。

>在比特币中，第一笔 coinbase 交易包含了如下信息：“The Times 03/Jan/2009 Chancellor on brink of second bailout for banks”。[可点击这里查看](https://blockchain.info/tx/4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b?show_adv=true).

`subsidy` 是挖出新块的奖励金。在比特币中，实际并没有存储这个数字，而是基于区块总数进行计算而得：区块总数除以 210000 就是 `subsidy`。挖出创世块的奖励是 50 BTC，每挖出 `210000` 个块后，奖励减半。在我们的实现中，这个奖励值将会是一个常量（至少目前是）。

## 将交易保存到区块链

从现在开始，每个块必须存储至少一笔交易。如果没有交易，也就不可能出新的块。这意味着我们应该移除 `Block` 的 `Data` 字段，取而代之的是存储交易：

```go
type Block struct {
	Timestamp     int64
	Transactions  []*Transaction
	PrevBlockHash []byte
	Hash          []byte
	Nonce         int
}
```

`NewBlock` 和 `NewGenesisBlock` 也必须做出相应改变：

```go
func NewBlock(transactions []*Transaction, prevBlockHash []byte) *Block {
	block := &Block{time.Now().Unix(), transactions, prevBlockHash, []byte{}, 0}
	...
}

func NewGenesisBlock(coinbase *Transaction) *Block {
	return NewBlock([]*Transaction{coinbase}, []byte{})
}
```

接下来修改创建区块链的函数：

```go
func CreateBlockchain(address string) *Blockchain {
	...
	err = db.Update(func(tx *bolt.Tx) error {
		cbtx := NewCoinbaseTX(address, genesisCoinbaseData)
		genesis := NewGenesisBlock(cbtx)

		b, err := tx.CreateBucket([]byte(blocksBucket))
		err = b.Put(genesis.Hash, genesis.Serialize())
		...
	})
	...
}
```

现在，这个函数会接受一个地址作为参数，这个地址将会被用来接收挖出创世块的奖励。

## 工作量证明

工作量证明算法必须要将存储在区块里面的交易考虑进去，从而保证区块链交易存储的一致性和可靠性。所以，我们必须修改 `ProofOfWork.prepareData` 方法：

```go
func (pow *ProofOfWork) prepareData(nonce int) []byte {
	data := bytes.Join(
		[][]byte{
			pow.block.PrevBlockHash,
			pow.block.HashTransactions(), // This line was changed
			IntToHex(pow.block.Timestamp),
			IntToHex(int64(targetBits)),
			IntToHex(int64(nonce)),
		},
		[]byte{},
	)

	return data
}
```

不像之前使用 `pow.block.Data`，现在我们使用 `pow.block.HashTransactions()` ：

```go
func (b *Block) HashTransactions() []byte {
	var txHashes [][]byte
	var txHash [32]byte

	for _, tx := range b.Transactions {
		txHashes = append(txHashes, tx.ID)
	}
	txHash = sha256.Sum256(bytes.Join(txHashes, []byte{}))

	return txHash[:]
}
```

通过哈希提供数据的唯一表示，这种做法我们已经不是第一次遇到了。我们想要通过仅仅一个哈希，就可以识别一个块里面的所有交易。为此，先获得每笔交易的哈希，然后将它们关联起来，最后获得一个连接后的组合哈希。

>比特币使用了一个更加复杂的技术：它将一个块里面包含的所有交易表示为一个  [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) ，然后在工作量证明系统中使用树的根哈希（root hash）。这个方法能够让我们快速检索一个块里面是否包含了某笔交易，即只需 root hash 而无需下载所有交易即可完成判断。

来检查一下到目前为止是否正确：

```bash
$ blockchain_go createblockchain -address Ivan
00000093450837f8b52b78c25f8163bb6137caf43ff4d9a01d1b731fa8ddcc8a

Done!
```

很好！我们已经获得了第一笔挖矿奖励，但是，我们要如何查看余额呢？
