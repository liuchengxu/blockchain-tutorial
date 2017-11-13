Part 6: 交易（2）
================

## 目录

<!-- vim-markdown-toc GFM -->

* [引言](#引言)
* [奖励](#奖励)
* [UTXO 集](#utxo-集)
* [Merkle 树](#merkle-树)
* [P2PKH](#p2pkh)
* [总结](#总结)

<!-- vim-markdown-toc -->

## 引言

在这个系列文章的一开始，我们就提到了，区块链是一个分布式数据库。不过在之前的文章中，我们选择性地跳过了“分布式”这个部分，而是将注意力都放到了“数据库”部分。到目前为止，我们几乎已经实现了一个区块链数据库的所有元素。今天，我们将会分析之前跳过的一些机制。而在下一篇文章中，我们将会开始讨论区块链的分布式特性。

之前的系列文章：

1. 基本原型
2. 工作量证明
3. 持久化和命令行接口
4. 交易（1）
5. 地址

>本文的代码实现变化很大，请点击 [这里](https://github.com/Jeiwan/blockchain_go/compare/part_5...part_6#files_bucket) 查看所有的代码更改。

## 奖励

在上一篇文章中，我们略过的一个小细节是挖矿奖励。现在，我们已经可以来完善这个细节了。

挖矿奖励，实际上就是一笔 coinbase 交易。当一个挖矿节点开始挖出一个新块时，它会将交易从队列中取出，并在前面附加一笔 coinbase 交易。coinbase 交易只有一个输出，里面包含了矿工的公钥哈希。

实现奖励，非常简单，更新 `send` 即可：

```go
func (cli *CLI) send(from, to string, amount int) {
    ...
    bc := NewBlockchain()
    UTXOSet := UTXOSet{bc}
    defer bc.db.Close()

    tx := NewUTXOTransaction(from, to, amount, &UTXOSet)
    cbTx := NewCoinbaseTX(from, "")
    txs := []*Transaction{cbTx, tx}

    newBlock := bc.MineBlock(txs)
    fmt.Println("Success!")
}
```

在我们的实现中，创建交易的人同时挖出了新块，所以会得到一笔奖励。

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

## Merkle 树

在这篇文章中，我还想要再讨论一个优化机制。

上如上面所提到的，完整的比特币数据库（也就是区块链）需要超过 140 Gb 的磁盘空间。因为比特币的去中心化特性，网络中的每个节点必须是独立，自给自足的，也就是每个节点必须存储一个区块链的完整副本。随着越来越多的人使用比特币，这条规则变得越来越难以遵守：因为不太可能每个人都去运行一个全节点。并且，由于节点是网络中的完全参与者，它们负有相关责任：节点必须验证交易和区块。另外，要想与其他节点交互和下载新块，也有一定的网络流量需求。

在中本聪的 [比特币原始论文](https://bitcoin.org/bitcoin.pdf) 中，对这个问题也有一个解决方案：简易支付验证（Simplified Payment Verification, SPV）。SPV 是一个比特币轻节点，它不需要下载整个区块链，也**不需要验证区块和交易**。相反，它会在区块链查找交易（为了验证支付），并且需要连接到一个全节点来检索必要的数据。这个机制允许在仅运行一个全节点的情况下有多个轻钱包。

为了实现 SPV，需要有一个方式来检查是否一个区块包含了某笔交易，而无须下载整个区块。这就是 Merkle 树所要完成的事情。

比特币用 Merkle 树来获取交易哈希，哈希被保存在区块头中，并会用于工作量证明系统。到目前为止，我们只是将一个块里面的每笔交易哈希连接了起来，将在上面应用了 SHA-256 算法。虽然这是一个用于获取区块交易唯一表示的一个不错的途径，但是它没有利用到 Merkle 树。

来看一下 Merkle 树：

![Merkle tree](http://upload-images.jianshu.io/upload_images/127313-9c708d3c3d6a19c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

每个块都会有一个 Merkle 树，它从叶子节点（树的底部）开始，一个叶子节点就是一个交易哈希（比特币使用双 SHA256 哈希）。叶子节点的数量必须是双数，但是并非每个块都包含了双数的交易。因为，如果一个块里面的交易数为单数，那么就将最后一个叶子节点（也就是 Merkle 树的最后一个交易，不是区块的最后一笔交易）复制一份凑成双数。

从下往上，两两成对，连接两个节点哈希，将组合哈希作为新的哈希。新的哈希就成为新的树节点。重复该过程，直到仅有一个节点，也就是树根。根哈希然后就会当做是整个块交易的唯一标示，将它保存到区块头，然后用于工作量证明。

Merkle 树的好处就是一个节点可以在不下载整个块的情况下，验证是否包含某笔交易。并且这些只需要一个交易哈希，一个 Merkle 树根哈希和一个 Merkle 路径。

最后，来写代码：

```go
type MerkleTree struct {
    RootNode *MerkleNode
}

type MerkleNode struct {
    Left  *MerkleNode
    Right *MerkleNode
    Data  []byte
}
```

先从结构体开始。每个 `MerkleNode` 包含数据和指向左右分支的指针。`MerkleTree` 实际上就是连接到下个节点的根节点，然后依次连接到更远的节点，等等。

让我们首先来创建一个新的节点：

```go
func NewMerkleNode(left, right *MerkleNode, data []byte) *MerkleNode {
    mNode := MerkleNode{}

    if left == nil && right == nil {
        hash := sha256.Sum256(data)
        mNode.Data = hash[:]
    } else {
        prevHashes := append(left.Data, right.Data...)
        hash := sha256.Sum256(prevHashes)
        mNode.Data = hash[:]
    }

    mNode.Left = left
    mNode.Right = right

    return &mNode
}
```

每个节点包含一些数据。当节点在叶子节点，数据从外界传入（在这里，也就是一个序列化后的交易）。当一个节点被关联到其他节点，它会将其他节点的数据取过来，连接后再哈希。

```go
func NewMerkleTree(data [][]byte) *MerkleTree {
    var nodes []MerkleNode

    if len(data)%2 != 0 {
        data = append(data, data[len(data)-1])
    }

    for _, datum := range data {
        node := NewMerkleNode(nil, nil, datum)
        nodes = append(nodes, *node)
    }

    for i := 0; i < len(data)/2; i++ {
        var newLevel []MerkleNode

        for j := 0; j < len(nodes); j += 2 {
            node := NewMerkleNode(&nodes[j], &nodes[j+1], nil)
            newLevel = append(newLevel, *node)
        }

        nodes = newLevel
    }

    mTree := MerkleTree{&nodes[0]}

    return &mTree
}
```

当生成一棵新树时，要确保的第一件事就是叶子节点必须是双数。然后，**数据**（也就是一个序列化后交易的数组）被转换成树的叶子，从这些叶子再慢慢形成一棵树。

现在，让我们来修改 `Block.HashTransactions`，它用于在工作量证明系统中获取交易哈希：

```go
func (b *Block) HashTransactions() []byte {
    var transactions [][]byte

    for _, tx := range b.Transactions {
        transactions = append(transactions, tx.Serialize())
    }
    mTree := NewMerkleTree(transactions)

    return mTree.RootNode.Data
}
```

首先，交易被序列化（使用 `encoding/gob`），然后使用序列后的交易构建一个 Mekle 树。树根将会作为块交易的唯一标识符。

## P2PKH

还有一件事情，我想要再谈一谈。

大家应该还记得，在比特币中有一个 *脚本（Script）*编程语言，它用于锁定交易输出；交易输入提供了解锁输出的数据。这个语言非常简单，用这个语言写的代码其实就是一系列数据和操作符而已。比如如下示例：

```
5 2 OP_ADD 7 OP_EQUAL
```

5, 2, 和 7 是数据，`OP_ADD` 和 `OP_EQUAL` 是操作符。*脚本*代码从左到右执行：将数据依次放入栈内，当遇到操作符时，就从栈内取出数据，并将操作符作用于数据，然后将结果作为栈顶元素。*脚本*的栈，实际上就是一个先进后出的内存存储：栈里的第一个元素最后一个取出，后面的每一个元素都会放到前一个元素之上。

让我们来对上面的脚本分部执行：

步骤   | 栈     | 脚本                    | 说明
:----: | :----  | :----                   | :----
1      | 空     | `5 2 OP_ADD 7 OP_EQUAL` | 一开始栈为空
2      | `5`    | `2 OP_ADD 7 OP_EQUAL`   | 从脚本里面取出 `5` 放入栈上
3      | `5 2`  | `OP_ADD 7 OP_EQUAL`     | 从脚本里面取出 `2` 放入栈上
4      | `7`    | `7 OP_EQUAL`            | 遇到操作符 `OP_ADD`, 从栈里取出两个操作数 `5` 和 `2`，相加后将结果放回栈上
5      | `7 7`  | `OP_EQUAL`              | 从脚本里面取出 `7` 放到栈上
6      | `true` | 空                      | 遇到操作符 `OP_EQUAL`，从栈里取出两个操作数并比较，将比较的结果放回栈内，脚本执行完毕，为空

`OP_ADD` 从栈内取两个元素，将这两个元素进行相加，然后将结果重新放回栈内。`OP_EQUAL` 从栈内取两个元素，然后对这两个元素进行比较：如果它们相等，就在栈上放一个 `true`，否则放一个 `false`。脚本执行的结果就是栈顶元素：在我们的案例中，如果是 `true`，那么表明脚本执行成功。

现在来看一下在比特币中，是如何用脚本执行支付的：

```
<signature> <pubKey> OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
```

这个脚本叫做 *Pay to Public Key Hash(P2PKH)*，这是比特币最常用的一个脚本。它所做的事情就是向一个公钥哈希支付，也就是说，用某一个公钥锁定一些币。这是**比特币支付的核心**：没有账户，没有资金转移；只有一个脚本检查提供的签名和公钥是否正确。

这个脚本实际存储为两个部分：

1. 第一个部分，`<signature> <pubkey>`，存储在输入的 `ScriptSig` 字段。

2. 第二部分，`OP_DUP OP_HASH160 <pubkeyHash> OP_EQUALVERYFY OP_CHECKSIG` 存储在输出的 `ScriptPubKey` 里面。

因此，输出定了解锁的逻辑，输入提供解锁输出的“钥匙”。然我们来执行一下这个脚本：

步骤   | 栈                                               | 脚本
:----: | :----                                            | :----
1      | 空                                               | `<signature> <pubKey> OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
2      | `<signature>`                                    | `<pubKey> OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
3      | `<signature> <pubkey>`                           | `OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
4      | `<signature> <pubKey> <pubKey>`                  | `OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
5      | `<signature> <pubKey> <pubKeyHash>`              | `<pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
6      | `<signature> <pubKey> <pubKeyHash> <pubKeyHash>` | `OP_EQUALVERIFY OP_CHECKSIG`
7      | `<signature> <pubKey>`                           | `OP_CHECKSIG`
8      | `true` 或 `false`                                | 空

`OP_DUP` 对栈顶元素进行复制。`OP_HASH160` 取栈顶元素，然后用 `RIPEMD160` 对它进行哈希，再将结果送回到栈上。`OP_EQUALVERIFY` 将栈顶的两个元素进行比较，如果它们不相等，终止脚本。`OP_CHECKSIG` 通过对交易进行哈希，并使用 `<signature>` 和 `pubKey` 来验证一笔交易的签名。最后的操作符有点复杂：它生成了一个修剪后的交易副本，对它进行哈希（因为它是一个被签名后的交易哈希），然后使用提供的 `<signature>` 和 `pubKey` 检查签名是否正确。

有了一个这样的脚本语言，实际上也可以让比特币成为一个智能合约平台：除了将一个单一的公钥转移资金，这个语言还使得一些其他的支付方案成为可能。

## 总结

这就是今天的全部内容了！我们已经实现了一个基于区块链的加密货币的几乎所有关键特性。我们已经有了区块链，地址，挖矿和交易。但是要想给这些所有的机制赋予生命，让比特币成为一个全球系统，还有一个不可或缺的环节：共识（consensus）。在下一篇文章中，我们将会开始实现区块链的“去中心化（decenteralized）”。敬请收听！

链接：

1. [Full source codes](https://github.com/Jeiwan/blockchain_go/tree/part_6)
2. [The UTXO Set](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage#The_UTXO_set_.28chainstate_leveldb.29)
3. [Merkle Tree](https://en.bitcoin.it/wiki/Protocol_documentation#Merkle_Trees)
4. [Script](https://en.bitcoin.it/wiki/Script)
5. [“Ultraprune” Bitcoin Core commit](https://github.com/sipa/bitcoin/commit/450cbb0944cd20a06ce806e6679a1f4c83c50db2)
6. [UTXO set statistics](https://statoshi.info/dashboard/db/unspent-transaction-output-set)
7. [Smart contracts and Bitcoin](https://medium.com/@maraoz/smart-contracts-and-bitcoin-a5d61011d9b1)
8. [Why every Bitcoin user should understand “SPV security”](https://medium.com/@jonaldfyookball/why-every-bitcoin-user-should-understand-spv-security-520d1d45e0b9)

原文链接：[Building Blockchain in Go. Part 6: Transactions 2](https://jeiwan.cc/posts/building-blockchain-in-go-part-6/)
