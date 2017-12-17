交易（2）
========

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
