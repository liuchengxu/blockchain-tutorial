交易（2）
========

## UTXO 集

在 Part 3: 持久化和命令行接口 中，我们研究了 Bitcoin Core 是如何在一个数据库中存储块的，并且了解到区块被存储在 `blocks` 数据库，交易输出被存储在 `chainstate` 数据库。会回顾一下 `chainstate` 的机构：

1. `c` + 32 字节的交易哈希 -> 该笔交易的未花费交易输出记录
2. `B` + 32 字节的块哈希 -> 未花费交易输出的块哈希

在之前那篇文章中，虽然我们已经实现了交易，但是并没有使用 `chainstate` 来存储交易的输出。所以，接下来我们继续完成这部分。

`chainstate` 不存储交易。它所存储的是 UTXO 集，也就是未花费交易输出的集合。除此以外，它还存储了“数据库表示的未花费交易输出的块哈希”，不过我们会暂时略过块哈希这一点，因为我们还没有用到块高度（但是我们会在接下来的文章中继续改进）。

那么，我们为什么需要 UTXO 集呢？

来思考一下我们早先实现的 `Blockchain.FindUnspentTransactions` 方法：

```go
func (bc *Blockchain) FindUnspentTransactions(pubKeyHash []byte) []Transaction {
    ...
    bci := bc.Iterator()

    for {
        block := bci.Next()

        for _, tx := range block.Transactions {
            ...
        }

        if len(block.PrevBlockHash) == 0 {
            break
        }
    }
    ...
}
```

这个函数找到有未花费输出的交易。由于交易被保存在区块中，所以它会对区块链里面的每一个区块进行迭代，检查里面的每一笔交易。截止 2017 年 9 月 18 日，在比特币中已经有 485，860 个块，整个数据库所需磁盘空间超过 140 Gb。这意味着一个人如果想要验证交易，必须要运行一个全节点。此外，验证交易将会需要在许多块上进行迭代。

整个问题的解决方案是有一个仅有未花费输出的索引，这就是 UTXO 集要做的事情：这是一个从所有区块链交易中构建（对区块进行迭代，但是只须做一次）而来的缓存，然后用它来计算余额和验证新的交易。截止 2017 年 9 月，UTXO 集大概有 2.7 Gb。

好了，让我们来想一下实现 UTXO 集的话需要作出哪些改变。目前，找到交易用到了以下一些方法：

1. `Blockchain.FindUnspentTransactions` - 找到有未花费输出交易的主要函数。也是在这个函数里面会对所有区块进行迭代。

2. `Blockchain.FindSpendableOutputs` - 这个函数用于当一个新的交易创建的时候。如果找到有所需数量的输出。使用 `Blockchain.FindUnspentTransactions`.

3. `Blockchain.FindUTXO` - 找到一个公钥哈希的未花费输出，然后用来获取余额。使用 `Blockchain.FindUnspentTransactions`.

4. `Blockchain.FindTransation` - 根据 ID 在区块链中找到一笔交易。它会在所有块上进行迭代直到找到它。

可以看到，所有方法都对数据库中的所有块进行迭代。但是目前我们还没有改进所有方法，因为 UTXO 集没法存储所有交易，只会存储那些有未花费输出的交易。因此，它无法用于 `Blockchain.FindTransaction`。

所以，我们想要以下方法：

1. `Blockchain.FindUTXO` - 通过对区块进行迭代找到所有未花费输出。

2. `UTXOSet.Reindex` - 使用 `UTXO` 找到未花费输出，然后在数据库中进行存储。这里就是缓存的地方。

3. `UTXOSet.FindSpendableOutputs` - 类似 `Blockchain.FindSpendableOutputs`，但是使用 UTXO 集。

4. `UTXOSet.FindUTXO` - 类似 `Blockchain.FindUTXO`，但是使用 UTXO 集。

5. `Blockchain.FindTransaction` 跟之前一样。

因此，从现在开始，两个最常用的函数将会使用 cache！来开始写代码吧。

```go
type UTXOSet struct {
    Blockchain *Blockchain
}
```

我们将会使用一个单一数据库，但是我们会将 UTXO 集从存储在不同的 bucket 中。因此，`UTXOSet` 跟 `Blockchain` 一起。

```go
func (u UTXOSet) Reindex() {
    db := u.Blockchain.db
    bucketName := []byte(utxoBucket)

    err := db.Update(func(tx *bolt.Tx) error {
        err := tx.DeleteBucket(bucketName)
        _, err = tx.CreateBucket(bucketName)
    })

    UTXO := u.Blockchain.FindUTXO()

    err = db.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket(bucketName)

        for txID, outs := range UTXO {
            key, err := hex.DecodeString(txID)
            err = b.Put(key, outs.Serialize())
        }
    })
}
```

这个方法初始化了 UTXO 集。首先，如果 bucket 存在就先移除，然后从区块链中获取所有的未花费输出，最终将输出保存到 bucket 中。

`Blockchain.FindUTXO` 几乎跟 `Blockchain.FindUnspentTransactions` 一模一样，但是现在它返回了一个 `TransactionID -> TransactionOutputs` 的 map。

现在，UTXO 集可以用于发送币：

```go
func (u UTXOSet) FindSpendableOutputs(pubkeyHash []byte, amount int) (int, map[string][]int) {
    unspentOutputs := make(map[string][]int)
    accumulated := 0
    db := u.Blockchain.db

    err := db.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoBucket))
        c := b.Cursor()

        for k, v := c.First(); k != nil; k, v = c.Next() {
            txID := hex.EncodeToString(k)
            outs := DeserializeOutputs(v)

            for outIdx, out := range outs.Outputs {
                if out.IsLockedWithKey(pubkeyHash) && accumulated < amount {
                    accumulated += out.Value
                    unspentOutputs[txID] = append(unspentOutputs[txID], outIdx)
                }
            }
        }
    })

    return accumulated, unspentOutputs
}
```

或者检查余额：

```go
func (u UTXOSet) FindUTXO(pubKeyHash []byte) []TXOutput {
    var UTXOs []TXOutput
    db := u.Blockchain.db

    err := db.View(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoBucket))
        c := b.Cursor()

        for k, v := c.First(); k != nil; k, v = c.Next() {
            outs := DeserializeOutputs(v)

            for _, out := range outs.Outputs {
                if out.IsLockedWithKey(pubKeyHash) {
                    UTXOs = append(UTXOs, out)
                }
            }
        }

        return nil
    })

    return UTXOs
}
```

这是 `Blockchain` 方法的简单修改后的版本。这个 `Blockchain` 方法已经不再需要了。

有了 UTXO 集，也就意味着我们的数据（交易）现在已经被分开存储：实际交易被存储在区块链中，未花费输出被存储在 UTXO 集中。这样一来，我们就需要一个良好的同步机制，因为我们想要 UTXO 集时刻处于最新状态，并且存储最新交易的输出。但是我们不想每生成一个新块，就重新生成索引，因为这正是我们要极力避免的频繁区块链扫描。因此，我们需要一个机制来更新 UTXO 集：

```go
func (u UTXOSet) Update(block *Block) {
    db := u.Blockchain.db

    err := db.Update(func(tx *bolt.Tx) error {
        b := tx.Bucket([]byte(utxoBucket))

        for _, tx := range block.Transactions {
            if tx.IsCoinbase() == false {
                for _, vin := range tx.Vin {
                    updatedOuts := TXOutputs{}
                    outsBytes := b.Get(vin.Txid)
                    outs := DeserializeOutputs(outsBytes)

                    for outIdx, out := range outs.Outputs {
                        if outIdx != vin.Vout {
                            updatedOuts.Outputs = append(updatedOuts.Outputs, out)
                        }
                    }

                    if len(updatedOuts.Outputs) == 0 {
                        err := b.Delete(vin.Txid)
                    } else {
                        err := b.Put(vin.Txid, updatedOuts.Serialize())
                    }

                }
            }

            newOutputs := TXOutputs{}
            for _, out := range tx.Vout {
                newOutputs.Outputs = append(newOutputs.Outputs, out)
            }

            err := b.Put(tx.ID, newOutputs.Serialize())
        }
    })
}
```

虽然这个方法看起来有点复杂，但是它所要做的事情非常直观。当挖出一个新块时，应该更新 UTXO 集。更新意味着移除已花费输出，并从新挖出来的交易中加入未花费输出。如果一笔交易的输出被移除，并且不再包含任何输出，那么这笔交易也应该被移除。相当简单！

现在让我们在必要的时候使用 UTXO 集：

```go
func (cli *CLI) createBlockchain(address string) {
    ...
    bc := CreateBlockchain(address)
    defer bc.db.Close()

    UTXOSet := UTXOSet{bc}
    UTXOSet.Reindex()
    ...
}
```

当一个新的区块链被创建以后，就会立刻进行重建索引。目前，这是 `Reindex` 唯一使用的地方，即使这里看起来有点“杀鸡用牛刀”，因为一条链开始的时候，只有一个块，里面只有一笔交易，`Update` 已经被使用了。不过我们在未来可能需要重建索引的机制。

```go
func (cli *CLI) send(from, to string, amount int) {
    ...
    newBlock := bc.MineBlock(txs)
    UTXOSet.Update(newBlock)
}
```

当挖出一个新块时，UTXO 集就会进行更新。

让我们来检查一下如否如期工作：

```go
$ blockchain_go createblockchain -address 1JnMDSqVoHi4TEFXNw5wJ8skPsPf4LHkQ1
00000086a725e18ed7e9e06f1051651a4fc46a315a9d298e59e57aeacbe0bf73

Done!

$ blockchain_go send -from 1JnMDSqVoHi4TEFXNw5wJ8skPsPf4LHkQ1 -to 12DkLzLQ4B3gnQt62EPRJGZ38n3zF4Hzt5 -amount 6
0000001f75cb3a5033aeecbf6a8d378e15b25d026fb0a665c7721a5bb0faa21b

Success!

$ blockchain_go send -from 1JnMDSqVoHi4TEFXNw5wJ8skPsPf4LHkQ1 -to 12ncZhA5mFTTnTmHq1aTPYBri4jAK8TacL -amount 4
000000cc51e665d53c78af5e65774a72fc7b864140a8224bf4e7709d8e0fa433

Success!

$ blockchain_go getbalance -address 1JnMDSqVoHi4TEFXNw5wJ8skPsPf4LHkQ1
Balance of '1F4MbuqjcuJGymjcuYQMUVYB37AWKkSLif': 20

$ blockchain_go getbalance -address 12DkLzLQ4B3gnQt62EPRJGZ38n3zF4Hzt5
Balance of '1XWu6nitBWe6J6v6MXmd5rhdP7dZsExbx': 6

$ blockchain_go getbalance -address 12ncZhA5mFTTnTmHq1aTPYBri4jAK8TacL
Balance of '13UASQpCR8Nr41PojH8Bz4K6cmTCqweskL': 4
```

很好！`1JnMDSqVoHi4TEFXNw5wJ8skPsPf4LHkQ1` 地址接收到了 3 笔奖励：

1. 一次是挖出创世块
2. 一次是挖出块 0000001f75cb3a5033aeecbf6a8d378e15b25d026fb0a665c7721a5bb0faa21b
3. 一个是挖出块 000000cc51e665d53c78af5e65774a72fc7b864140a8224bf4e7709d8e0fa433


