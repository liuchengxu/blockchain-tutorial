交易（1）
========

## 交易输出

让我们先从输出（output）开始：

```go
type TXOutput struct {
	Value        int
	ScriptPubKey string
}
```

实际上，正是输出里面存储了“币”（注意，也就是上面的 `Value` 字段）。而这里的存储，指的是用一个数学难题对输出进行锁定，这个难题被存储在 `ScriptPubKey` 里面。在内部，比特币使用了一个叫做 *Script* 的脚本语言，用它来定义锁定和解锁输出的逻辑。虽然这个语言相当的原始（这是为了避免潜在的黑客攻击和滥用而有意为之），并不复杂，但是我们也并不会在这里讨论它的细节。你可以在[这里](https://en.bitcoin.it/wiki/Script) 找到详细解释。

>在比特币中，`value` 字段存储的是 *satoshi* 的数量，而不是 BTC 的数量。一个 *satoshi* 等于一百万分之一的 BTC(0.00000001 BTC)，这也是比特币里面最小的货币单位（就像是 1 分的硬币）。

由于还没有实现地址（address），所以目前我们会避免涉及逻辑相关的完整脚本。`ScriptPubKey` 将会存储一个任意的字符串（用户定义的钱包地址）。

>顺便说一下，有了一个这样的脚本语言，也意味着比特币其实也可以作为一个智能合约平台。

关于输出，非常重要的一点是：它们是**不可再分的（indivisible）**。也就是说，你无法仅引用它的其中某一部分。要么不用，如果要用，必须一次性用完。当一个新的交易中引用了某个输出，那么这个输出必须被全部花费。如果它的值比需要的值大，那么就会产生一个找零，找零会返还给发送方。这跟现实世界的场景十分类似，当你想要支付的时候，如果一个东西值 1 美元，而你给了一个 5 美元的纸币，那么你会得到一个 4 美元的找零。

## 发送币

现在，我们想要给其他人发送一些币。为此，我们需要创建一笔新的交易，将它放到一个块里，然后挖出这个块。之前我们只实现了 coinbase 交易（这是一种特殊的交易），现在我们需要一种通用的普通交易：

```go
func NewUTXOTransaction(from, to string, amount int, bc *Blockchain) *Transaction {
	var inputs []TXInput
	var outputs []TXOutput

	acc, validOutputs := bc.FindSpendableOutputs(from, amount)

	if acc < amount {
		log.Panic("ERROR: Not enough funds")
	}

	// Build a list of inputs
	for txid, outs := range validOutputs {
		txID, err := hex.DecodeString(txid)

		for _, out := range outs {
			input := TXInput{txID, out, from}
			inputs = append(inputs, input)
		}
	}

	// Build a list of outputs
	outputs = append(outputs, TXOutput{amount, to})
	if acc > amount {
		outputs = append(outputs, TXOutput{acc - amount, from}) // a change
	}

	tx := Transaction{nil, inputs, outputs}
	tx.SetID()

	return &tx
}
```

在创建新的输出前，我们首先必须找到所有的未花费输出，并且确保它们有足够的价值（value），这就是 `FindSpendableOutputs` 方法要做的事情。随后，对于每个找到的输出，会创建一个引用该输出的输入。接下来，我们创建两个输出：

1. 一个由接收者地址锁定。这是给其他地址实际转移的币。

2. 一个由发送者地址锁定。这是一个找零。只有当未花费输出超过新交易所需时产生。记住：输出是**不可再分的**。

`FindSpendableOutputs` 方法基于之前定义的 `FindUnspentTransactions` 方法：

```go
func (bc *Blockchain) FindSpendableOutputs(address string, amount int) (int, map[string][]int) {
	unspentOutputs := make(map[string][]int)
	unspentTXs := bc.FindUnspentTransactions(address)
	accumulated := 0

Work:
	for _, tx := range unspentTXs {
		txID := hex.EncodeToString(tx.ID)

		for outIdx, out := range tx.Vout {
			if out.CanBeUnlockedWith(address) && accumulated < amount {
				accumulated += out.Value
				unspentOutputs[txID] = append(unspentOutputs[txID], outIdx)

				if accumulated >= amount {
					break Work
				}
			}
		}
	}

	return accumulated, unspentOutputs
}
```

这个方法对所有的未花费交易进行迭代，并对它的值进行累加。当累加值大于或等于我们想要传送的值时，它就会停止并返回累加值，同时返回的还有通过交易 ID 进行分组的输出索引。我们只需取出足够支付的钱就够了。

现在，我们可以修改 `Blockchain.MineBlock` 方法：

```go
func (bc *Blockchain) MineBlock(transactions []*Transaction) {
	...
	newBlock := NewBlock(transactions, lastHash)
	...
}
```

最后，让我们来实现 `send` 方法：

```go
func (cli *CLI) send(from, to string, amount int) {
	bc := NewBlockchain(from)
	defer bc.db.Close()

	tx := NewUTXOTransaction(from, to, amount, bc)
	bc.MineBlock([]*Transaction{tx})
	fmt.Println("Success!")
}
```

发送币意味着创建新的交易，并通过挖出新块的方式将交易打包到区块链中。不过，比特币并不是一连串立刻完成这些事情（虽然我们目前的实现是这么做的）。相反，它会将所有新的交易放到一个内存池中（mempool），然后当矿工准备挖出一个新块时，它就从内存池中取出所有交易，创建一个候选块。只有当包含这些交易的块被挖出来，并添加到区块链以后，里面的交易才开始确认。

让我们来检查一下发送币是否能工作：

```bash
$ blockchain_go send -from Ivan -to Pedro -amount 6
00000001b56d60f86f72ab2a59fadb197d767b97d4873732be505e0a65cc1e37

Success!

$ blockchain_go getbalance -address Ivan
Balance of 'Ivan': 4

$ blockchain_go getbalance -address Pedro
Balance of 'Pedro': 6
```

很好！现在，让我们创建更多的交易，确保从多个输出中发送币也正常工作：

```bash
$ blockchain_go send -from Pedro -to Helen -amount 2
00000099938725eb2c7730844b3cd40209d46bce2c2af9d87c2b7611fe9d5bdf

Success!

$ blockchain_go send -from Ivan -to Helen -amount 2
000000a2edf94334b1d94f98d22d7e4c973261660397dc7340464f7959a7a9aa

Success!
```

现在，Helen 的币被锁定在了两个输出中：一个来自 Pedro，一个来自 Ivan。让我们把它们发送给其他人：

```bash
$ blockchain_go send -from Helen -to Rachel -amount 3
000000c58136cffa669e767b8f881d16e2ede3974d71df43058baaf8c069f1a0

Success!

$ blockchain_go getbalance -address Ivan
Balance of 'Ivan': 2

$ blockchain_go getbalance -address Pedro
Balance of 'Pedro': 4

$ blockchain_go getbalance -address Helen
Balance of 'Helen': 1

$ blockchain_go getbalance -address Rachel
Balance of 'Rachel': 3
```

看起来没问题！现在，来测试一些失败的情况：

```bash
$ blockchain_go send -from Pedro -to Ivan -amount 5
panic: ERROR: Not enough funds

$ blockchain_go getbalance -address Pedro
Balance of 'Pedro': 4

$ blockchain_go getbalance -address Ivan
Balance of 'Ivan': 2
```

## 总结

虽然不容易，但是现在终于实现交易了！不过，我们依然缺少了一些像比特币那样的一些关键特性：

1. 地址（address）。我们还没有基于私钥（private key）的真实地址。

2. 奖励（reward）。现在挖矿是肯定无法盈利的！

3. UTXO 集。获取余额需要扫描整个区块链，而当区块非常多的时候，这么做就会花费很长时间。并且，如果我们想要验证后续交易，也需要花费很长时间。而 UTXO 集就是为了解决这些问题，加快交易相关的操作。

4. 内存池（mempool）。在交易被打包到块之前，这些交易被存储在内存池里面。在我们目前的实现中，一个块仅仅包含一笔交易，这是相当低效的。

链接:

1. [Full source codes](https://github.com/Jeiwan/blockchain_go/tree/part_4)

2. [Transaction](https://en.bitcoin.it/wiki/Transaction)

3. [Merkle tree](https://en.bitcoin.it/wiki/Protocol_documentation#Merkle_Trees)

4. [Coinbase](https://en.bitcoin.it/wiki/Coinbase)

本文源代码：[part_4](https://github.com/Jeiwan/blockchain_go/tree/part_4)

原文链接：[Building Blockchain in Go. Part 4: Transactions 1](https://jeiwan.cc/posts/building-blockchain-in-go-part-4/)
