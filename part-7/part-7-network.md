Part 7: 网络
============

## 目录

<!-- vim-markdown-toc GFM -->

* [引言](#引言)
* [区块链网络](#区块链网络)
* [节点角色](#节点角色)
* [网络简化](#网络简化)
* [实现](#实现)
* [场景](#场景)
* [版本](#版本)
* [getblocks](#getblocks)
* [inv](#inv)
* [getdata](#getdata)
* [block 和 tx](#block-和-tx)
* [结果](#结果)
  * [NODE 3000](#node-3000)
  * [NODE 3001](#node-3001)
  * [NODE 3000](#node-3000-1)
  * [NODE 3001](#node-3001-1)
  * [NODE 3002](#node-3002)
  * [NODE 3001](#node-3001-2)
  * [NODE 3002](#node-3002-1)
  * [NODE 3001](#node-3001-3)
* [总结](#总结)

<!-- vim-markdown-toc -->

## 引言

到目前为止，我们所构建的原型已经具备了区块链所有的关键特性：匿名，安全，随机生成的地址；区块链数据存储；工作量证明系统；可靠地存储交易。尽管这些特性都不可或缺，但是仍有不足。能够使得这些特性真正发光发热，使得加密货币成为可能的，是**网络（network）**。如果实现的这样一个区块链仅仅运行在单一节点上，有什么用呢？如果只有一个用户，那么这些基于密码学的特性，又有什么用呢？正是由于网络，才使得整个机制能够运转和发光发热。

你可以将这些区块链特性认为是规则（rule），类似于人类在一起生活，繁衍生息建立的规则，一种社会安排。区块链网络就是一个程序社区，里面的每个程序都遵循同样的规则，正是由于遵循着同一个规则，才使得网络能够长存。类似的，当人们都有着同样的想法，就能够将拳头攥在一起构建一个更好的生活。如果有人遵循着不同的规则，那么他们就将生活在一个分裂的社区（州，公社，等等）中。同样的，如果有区块链节点遵循不同的规则，那么也会形成一个分裂的网络。

**重点在于**：如果没有网络，或者大部分节点都不遵守同样的规则，那么规则就会形同虚设，毫无用处！

>声明：不幸的是，我并没有足够的时间来实现一个真实的 P2P 网络原型。本文我会展示一个最常见的场景，这个场景涉及不同类型的节点。继续改进这个场景，将它实现为一个 P2P 网络，对你来说是一个很好的挑战和实践！除了本文的场景，我也无法保证在其他场景将会正常工作。抱歉！
>
>本文的代码实现变化很大，请点击 [这里](https://github.com/Jeiwan/blockchain_go/compare/part_6...part_7#files_bucket) 查看所有的代码更改。

## 区块链网络

区块链网络是去中心化的，这意味着没有服务器，客户端也不需要依赖服务器来获取或处理数据。在区块链网络中，有的是节点，每个节点是网络的一个完全（full-fledged）成员。节点就是一切：它既是一个客户端，也是一个服务器。这一点需要牢记于心，因为这与传统的网页应用非常不同。

区块链网络是一个 P2P（Peer-to-Peer，端到端）的网络，即节点直接连接到其他节点。它的拓扑是扁平的，因为在节点的世界中没有层级之分。下面是它的示意图：

![schematic representation](http://upload-images.jianshu.io/upload_images/127313-3c4174ad7cf6edf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[Business vector created by Dooder - Freepik.com](http://www.freepik.com/dooder)

要实现这样一个网络节点更加困难，因为它们必须执行很多操作。每个节点必须与很多其他节点进行交互，它必须请求其他节点的状态，与自己的状态进行比较，当状态过时时进行更新。

## 节点角色

尽管节点具有完备成熟的属性，但是它们也可以在网络中扮演不同角色。比如：

1. 矿工
    这样的节点运行于强大或专用的硬件（比如 ASIC）之上，它们唯一的目标是，尽可能快地挖出新块。矿工是区块链中唯一可能会用到工作量证明的角色，因为挖矿实际上意味着解决 PoW 难题。在权益证明 PoS 的区块链中，没有挖矿。

2. 全节点
    这些节点验证矿工挖出来的块的有效性，并对交易进行确认。为此，他们必须拥有区块链的完整拷贝。同时，全节点执行路由操作，帮助其他节点发现彼此。对于网络来说，非常重要的一段就是要有足够多的全节点。因为正是这些节点执行了决策功能：他们决定了一个块或一笔交易的有效性。

3. SPV
    SPV 表示 Simplified Payment Verification，简单支付验证。这些节点并不存储整个区块链副本，但是仍然能够对交易进行验证（不过不是验证全部交易，而是一个交易子集，比如，发送到某个指定地址的交易）。一个 SPV 节点依赖一个全节点来获取数据，可能有多个 SPV 节点连接到一个全节点。SPV 使得钱包应用成为可能：一个人不需要下载整个区块链，但是仍能够验证他的交易。

## 网络简化

为了在目前的区块链原型中实现网络，我们不得不简化一些事情。因为我们没有那么多的计算机来模拟一个多节点的网络。当然，我们可以使用虚拟机或是 Docker 来解决这个问题，但是这会使一切都变得更复杂：你将不得不先解决可能出现的虚拟机或 Docker 问题，而我的目标是将全部精力都放在区块链实现上。所以，我们想要在一台机器上运行多个区块链节点，同时希望它们有不同的地址。为了实现这一点，我们将使用**端口号作为节点标识符**，而不是使用 IP 地址，比如将会有这样地址的节点：**127.0.0.1:3000**，**127.0.0.1:3001**，**127.0.0.1:3002** 等等。我们叫它端口节点（port node） ID，并使用环境变量 `NODE_ID` 对它们进行设置。故而，你可以打开多个终端窗口，设置不同的 `NODE_ID` 运行不同的节点。

这个方法也需要有不同的区块链和钱包文件。它们现在必须依赖于节点 ID 进行命名，比如 blockchain_3000.db, blockchain_30001.db and wallet_3000.db, wallet_30001.db 等等。

## 实现

所以，当你下载 Bitcoin Core 并首次运行时，到底发生了什么呢？它必须连接到某个节点下载最新状态的区块链。考虑到你的电脑并没有意识到所有或是部分的比特币节点，那么连接到的“某个节点”到底是什么？

在 Bitcoin Core 中硬编码一个地址，已经被证实是一个错误：因为节点可能会被攻击或关机，这会导致新的节点无法加入到网络中。在 Bitcoin Core 中，硬编码了 [DNS seeds](https://bitcoin.org/en/glossary/dns-seed)。虽然这些并不是节点，但是 DNS 服务器知道一些节点的地址。当你启动一个全新的 Bitcoin Core 时，它会连接到一个种子节点，获取全节点列表，随后从这些节点中下载区块链。

不过在我们目前的实现中，无法做到完全的去中心化，因为会出现中心化的特点。我们会有三个节点：

1. 一个中心节点。所有其他节点都会连接到这个节点，这个节点会在其他节点之间发送数据。

2. 一个矿工节点。这个节点会在内存池中存储新的交易，当有足够的交易时，它就会打包挖出一个新块。

3. 一个钱包节点。这个节点会被用作在钱包之间发送币。但是与 SPV 节点不同，它存储了区块链的一个完整副本。

## 场景

本文的目标是实现如下场景：

1. 中心节点创建一个区块链。
2. 一个其他（钱包）节点连接到中心节点并下载区块链。
3. 另一个（矿工）节点连接到中心节点并下载区块链。
4. 钱包节点创建一笔交易。
5. 矿工节点接收交易，并将交易保存到内存池中。
6. 当内存池中有足够的交易时，矿工开始挖一个新块。
7. 当挖出一个新块后，将其发送到中心节点。
8. 钱包节点与中心节点进行同步。
9. 钱包节点的用户检查他们的支付是否成功。

这就是比特币中的一般流程。尽管我们不会实现一个真实的 P2P 网络，但是我们会实现一个真是，也是比特币最常见最重要的用户场景。

## 版本

节点通过消息（message）进行交流。当一个新的节点开始运行时，它会从一个 DNS 种子获取几个节点，给它们发送 `version` 消息，在我们的实现看起来就像是这样：

```go
type version struct {
    Version    int
    BestHeight int
    AddrFrom   string
}
```

由于我们仅有一个区块链版本，所以 `Version` 字段实际并不会存储什么重要信息。`BestHeight` 存储区块链中节点的高度。`AddFrom` 存储发送者的地址。

接收到 `version` 消息的节点应该做什么呢？它会响应自己的 `version` 消息。这是一种握手🤝：如果没有事先互相问候，就不可能有其他交流。不过，这并不是处于礼貌：`version` 用于找到一个更长的区块链。当一个节点接收到 `version` 消息，它会检查本节点的区块链是否比 `BestHeight` 的值更大。如果不是，节点就会请求并下载缺失的块。

为了接收消息，我们需要一个服务器：

```go
var nodeAddress string
var knownNodes = []string{"localhost:3000"}

func StartServer(nodeID, minerAddress string) {
    nodeAddress = fmt.Sprintf("localhost:%s", nodeID)
    miningAddress = minerAddress
    ln, err := net.Listen(protocol, nodeAddress)
    defer ln.Close()

    bc := NewBlockchain(nodeID)

    if nodeAddress != knownNodes[0] {
        sendVersion(knownNodes[0], bc)
    }

    for {
        conn, err := ln.Accept()
        go handleConnection(conn, bc)
    }
}
```

首先，我们对中心节点的地址进行硬编码：因为每个节点必须知道从何处开始初始化。`minerAddress` 参数指定了接收挖矿奖励的地址。代码片段：

```go
if nodeAddress != knownNodes[0] {
    sendVersion(knownNodes[0], bc)
}
```

这意味着如果当前节点不是中心节点，它必须向中心节点发送 `version` 消息来查询是否自己的区块链已过时。

```go
func sendVersion(addr string, bc *Blockchain) {
    bestHeight := bc.GetBestHeight()
    payload := gobEncode(version{nodeVersion, bestHeight, nodeAddress})

    request := append(commandToBytes("version"), payload...)

    sendData(addr, request)
}
```

我们的消息，在底层就是字节序列。前 12 个字节指定了命令名（比如这里的 `version`），后面的字节会包含 **gob** 编码的消息结构，`commandToBytes` 看起来是这样：

```go
func commandToBytes(command string) []byte {
    var bytes [commandLength]byte

    for i, c := range command {
        bytes[i] = byte(c)
    }

    return bytes[:]
}
```

它创建一个 12 字节的缓冲区，并用命令名进行填充，将剩下的字节置为空。下面一个相反的函数：

```go
func bytesToCommand(bytes []byte) string {
    var command []byte

    for _, b := range bytes {
        if b != 0x0 {
            command = append(command, b)
        }
    }

    return fmt.Sprintf("%s", command)
}
```

当一个节点接收到一个命令，它会运行 `bytesToCommand` 来提取命令名，并选择正确的处理器处理命令主体：

```go
func handleConnection(conn net.Conn, bc *Blockchain) {
    request, err := ioutil.ReadAll(conn)
    command := bytesToCommand(request[:commandLength])
    fmt.Printf("Received %s command\n", command)

    switch command {
    ...
    case "version":
        handleVersion(request, bc)
    default:
        fmt.Println("Unknown command!")
    }

    conn.Close()
}
```

下面是 `version` 命令处理器：

```go
func handleVersion(request []byte, bc *Blockchain) {
    var buff bytes.Buffer
    var payload verzion

    buff.Write(request[commandLength:])
    dec := gob.NewDecoder(&buff)
    err := dec.Decode(&payload)

    myBestHeight := bc.GetBestHeight()
    foreignerBestHeight := payload.BestHeight

    if myBestHeight < foreignerBestHeight {
        sendGetBlocks(payload.AddrFrom)
    } else if myBestHeight > foreignerBestHeight {
        sendVersion(payload.AddrFrom, bc)
    }

    if !nodeIsKnown(payload.AddrFrom) {
        knownNodes = append(knownNodes, payload.AddrFrom)
    }
}
```

首先，我们需要对请求进行解码，提取有效信息。所有的处理器在这部分都类似，所以我们会下面的代码片段中略去这部分。

然后节点将从消息中提取的 `BestHeight` 与自身进行比较。如果自身节点的区块链更长，它会回复 `version` 消息；否则，它会发送 `getblocks` 消息。

## getblocks

```go
type getblocks struct {
    AddrFrom string
}
```

`getblocks` 意为 “给我看一下你有什么区块”（在比特币中，这会更加复杂）。注意，它并没有说“把你全部的区块给我”，而是请求了一个块哈希的列表。这是为了减轻网络负载，因为区块可以从不同的节点下载，并且我们不想从一个单一节点下载数十 GB 的数据。

处理命令十分简单：

```go
func handleGetBlocks(request []byte, bc *Blockchain) {
    ...
    blocks := bc.GetBlockHashes()
    sendInv(payload.AddrFrom, "block", blocks)
}
```

在我们简化版的实现中，它会返回 **所有块哈希**。

## inv

```go
type inv struct {
    AddrFrom string
    Type     string
    Items    [][]byte
}
```

比特币使用 `inv` 来向其他节点展示当前节点有什么块和交易。再次提醒，它没有包含完整的区块链和交易，仅仅是哈希而已。`Type` 字段表明了这是块还是交易。

处理 `inv` 稍显复杂：

```go
func handleInv(request []byte, bc *Blockchain) {
    ...
    fmt.Printf("Recevied inventory with %d %s\n", len(payload.Items), payload.Type)

    if payload.Type == "block" {
        blocksInTransit = payload.Items

        blockHash := payload.Items[0]
        sendGetData(payload.AddrFrom, "block", blockHash)

        newInTransit := [][]byte{}
        for _, b := range blocksInTransit {
            if bytes.Compare(b, blockHash) != 0 {
                newInTransit = append(newInTransit, b)
            }
        }
        blocksInTransit = newInTransit
    }

    if payload.Type == "tx" {
        txID := payload.Items[0]

        if mempool[hex.EncodeToString(txID)].ID == nil {
            sendGetData(payload.AddrFrom, "tx", txID)
        }
    }
}
```

如果收到块哈希，我们想要将它们保存在 `blocksInTransit` 变量来跟踪已下载的块。这能够让我们从不同的节点下载块。在将块置于传送状态时，我们给 `inv` 消息的发送者发送 `getdata` 命令并更新 `blocksInTransit`。在一个真实的 P2P 网络中，我们会想要从不同节点来传送块。

在我们的实现中，我们永远也不会发送有多重哈希的 `inv`。这就是为什么当 `payload.Type == "tx"` 时，只会拿到第一个哈希。然后我们检查是否在内存池中已经有了这个哈希，如果没有，发送 `getdata` 消息。

## getdata

```go
type getdata struct {
    AddrFrom string
    Type     string
    ID       []byte
}
```

`getdata` 用于某个块或交易的请求，它可以仅包含一个块或交易的 ID。

```go
func handleGetData(request []byte, bc *Blockchain) {
    ...
    if payload.Type == "block" {
        block, err := bc.GetBlock([]byte(payload.ID))

        sendBlock(payload.AddrFrom, &block)
    }

    if payload.Type == "tx" {
        txID := hex.EncodeToString(payload.ID)
        tx := mempool[txID]

        sendTx(payload.AddrFrom, &tx)
    }
}
```

这个处理器比较地直观：如果它们请求一个块，则返回块；如果它们请求一笔交易，则返回交易。注意，我们并不检查实际上是否已经有了这个块或交易。这是一个缺陷 :)

## block 和 tx

```go
type block struct {
    AddrFrom string
    Block    []byte
}

type tx struct {
    AddFrom     string
    Transaction []byte
}
```

实际完成数据转移的正是这些消息。

处理 `block` 消息十分简单：

```go
func handleBlock(request []byte, bc *Blockchain) {
    ...

    blockData := payload.Block
    block := DeserializeBlock(blockData)

    fmt.Println("Recevied a new block!")
    bc.AddBlock(block)

    fmt.Printf("Added block %x\n", block.Hash)

    if len(blocksInTransit) > 0 {
        blockHash := blocksInTransit[0]
        sendGetData(payload.AddrFrom, "block", blockHash)

        blocksInTransit = blocksInTransit[1:]
    } else {
        UTXOSet := UTXOSet{bc}
        UTXOSet.Reindex()
    }
}
```

当接收到一个新块时，我们把它放到区块链里面。如果还有更多的区块需要下载，我们继续从上一个下载的块的那个节点继续请求。当最后把所有块都下载完后，对 UTXO 集进行重新索引。

>TODO：并非无条件信任，我们应该在将每个块加入到区块链之前对它们进行验证。
>
>TODO: 并非运行 UTXOSet.Reindex()， 而是应该使用 UTXOSet.Update(block)，因为如果区块链很大，它将需要很多时间来对整个 UTXO 集重新索引。

处理 `tx` 消息是最困难的部分：

```go
func handleTx(request []byte, bc *Blockchain) {
    ...
    txData := payload.Transaction
    tx := DeserializeTransaction(txData)
    mempool[hex.EncodeToString(tx.ID)] = tx

    if nodeAddress == knownNodes[0] {
        for _, node := range knownNodes {
            if node != nodeAddress && node != payload.AddFrom {
                sendInv(node, "tx", [][]byte{tx.ID})
            }
        }
    } else {
        if len(mempool) >= 2 && len(miningAddress) > 0 {
        MineTransactions:
            var txs []*Transaction

            for id := range mempool {
                tx := mempool[id]
                if bc.VerifyTransaction(&tx) {
                    txs = append(txs, &tx)
                }
            }

            if len(txs) == 0 {
                fmt.Println("All transactions are invalid! Waiting for new ones...")
                return
            }

            cbTx := NewCoinbaseTX(miningAddress, "")
            txs = append(txs, cbTx)

            newBlock := bc.MineBlock(txs)
            UTXOSet := UTXOSet{bc}
            UTXOSet.Reindex()

            fmt.Println("New block is mined!")

            for _, tx := range txs {
                txID := hex.EncodeToString(tx.ID)
                delete(mempool, txID)
            }

            for _, node := range knownNodes {
                if node != nodeAddress {
                    sendInv(node, "block", [][]byte{newBlock.Hash})
                }
            }

            if len(mempool) > 0 {
                goto MineTransactions
            }
        }
    }
}
```

首先要做的事情是将新交易放到内存池中（再次提醒，在将交易放到内存池之前，必要对其进行验证）。下个片段：

```go
if nodeAddress == knownNodes[0] {
    for _, node := range knownNodes {
        if node != nodeAddress && node != payload.AddFrom {
            sendInv(node, "tx", [][]byte{tx.ID})
        }
    }
}
```

检查当前节点是否是中心节点。在我们的实现中，中心节点并不会挖矿。它只会将新的交易推送给网络中的其他节点。

下一个很大的代码片段是矿工节点“专属”。让我们对它进行一下分解:

```go
if len(mempool) >= 2 && len(miningAddress) > 0 {
```

`miningAddress` 只会在矿工节点上设置。如果当前节点（矿工）的内存池中有两笔或更多的交易，开始挖矿：

```go
for id := range mempool {
    tx := mempool[id]
    if bc.VerifyTransaction(&tx) {
        txs = append(txs, &tx)
    }
}

if len(txs) == 0 {
    fmt.Println("All transactions are invalid! Waiting for new ones...")
    return
}
```

首先，内存池中所有交易都是通过验证的。无效的交易会被忽略，如果没有有效交易，则挖矿中断。

```go
cbTx := NewCoinbaseTX(miningAddress, "")
txs = append(txs, cbTx)

newBlock := bc.MineBlock(txs)
UTXOSet := UTXOSet{bc}
UTXOSet.Reindex()

fmt.Println("New block is mined!")
```

验证后的交易被放到一个块里，同时还有附带奖励的 coinbase 交易。当块被挖出来以后，UTXO 集会被重新索引。

>TODO: 提醒，应该使用 UTXOSet.Update 而不是 UTXOSet.Reindex.

```go
for _, tx := range txs {
    txID := hex.EncodeToString(tx.ID)
    delete(mempool, txID)
}

for _, node := range knownNodes {
    if node != nodeAddress {
        sendInv(node, "block", [][]byte{newBlock.Hash})
    }
}

if len(mempool) > 0 {
    goto MineTransactions
}
```

当一笔交易被挖出来以后，就会被从内存池中移除。当前节点所连接到的所有其他节点，接收带有新块哈希的 `inv` 消息。在处理完消息后，它们可以对块进行请求。

## 结果

让我们来回顾一下上面定义的场景。

首先，在第一个终端窗口中将 `NODE_ID` 设置为 3000（`export NODE_ID=3000`）。为了让你知道什么节点执行什么操作，我会使用像 **NODE 3000** 或 **NODE 3001** 进行标识。

### NODE 3000

创建一个钱包和一个新的区块链：

```bash
$ blockchain_go createblockchain -address CENTREAL_NODE
```
（为了简洁起见，我会使用假地址。）

然后，会生成一个仅包含创世块的区块链。我们需要保存块，并在其他节点使用。创世块承担了一条链标识符的角色（在 Bitcoin Core 中，创世块是硬编码的）

```bash
$ cp blockchain_3000.db blockchain_genesis.db 
```

### NODE 3001

接下来，打开一个新的终端窗口，将 node ID 设置为 3001。这会作为一个钱包节点。通过 `blockchain_go createwallet` 生成一些地址，我们把这些地址叫做  WALLET_1, WALLET_2, WALLET_3.

### NODE 3000

向钱包地址发送一些币：

```bash
$ blockchain_go send -from CENTREAL_NODE -to WALLET_1 -amount 10 -mine
$ blockchain_go send -from CENTREAL_NODE -to WALLET_2 -amount 10 -mine
```

`-mine` 标志指的是块会立刻被同一节点挖出来。我们必须要有这个标志，因为初始状态时，网络中没有矿工节点。

启动节点：

```bash
$ blockchain_go startnode
```

这个节点会持续运行，直到本文定义的场景结束。

### NODE 3001

启动上面保存创世块节点的区块链：

```bash
$ cp blockchain_genesis.db blockchain_3001.db
```

运行节点：

```go
$ blockchain_go startnode
```

它会从中心节点下载所有区块。为了检查一切正常，暂停节点运行并检查余额：

```bash
$ blockchain_go getbalance -address WALLET_1
Balance of 'WALLET_1': 10

$ blockchain_go getbalance -address WALLET_2
Balance of 'WALLET_2': 10
```

你还可以检查 `CENTRAL_NODE` 地址的余额，因为 node 3001 现在有它自己的区块链：

```bash
$ blockchain_go getbalance -address CENTRAL_NODE
Balance of 'CENTRAL_NODE': 10
```

### NODE 3002

打开一个新的终端窗口，将它的 ID 设置为 3002，然后生成一个钱包。这会是一个矿工节点。初始化区块链：

```bash
$ cp blockchain_genesis.db blockchain_3002.db
```

启动节点：

```bash
$ blockchain_go startnode -miner MINER_WALLET
```

### NODE 3001

发送一些币：

```bash
$ blockchain_go send -from WALLET_1 -to WALLET_3 -amount 1
$ blockchain_go send -from WALLET_2 -to WALLET_4 -amount 1
```

### NODE 3002

迅速切换到矿工节点，你会看到挖出了一个新块！同时，检查中心节点的输出。

### NODE 3001

切换到钱包节点并启动：

```bash
$ blockchain_go startnode
```

它会下载最近挖出来的块！

暂停节点并检查余额：

```bash
$ blockchain_go getbalance -address WALLET_1
Balance of 'WALLET_1': 9

$ blockchain_go getbalance -address WALLET_2
Balance of 'WALLET_2': 9

$ blockchain_go getbalance -address WALLET_3
Balance of 'WALLET_3': 1

$ blockchain_go getbalance -address WALLET_4
Balance of 'WALLET_4': 1

$ blockchain_go getbalance -address MINER_WALLET
Balance of 'MINER_WALLET': 10
```

就是这么多了！

## 总结

这是本系列的最后一篇文章了。我本可以就实现一个真实的 P2P 网络原型继续展开，但是我真的没有这么多时间。我希望本文已经回答了关于比特币技术的一些问题，也给读者提出了一些问题，这些é®题你可以自行寻找答案。在比特币技术中还有隐藏着很多有趣的事情！好运！

后记：你可以从实现 `addr` 消息来开始改进网络，正如比特币网络协议中所描述的（链接可以下方找到）那样。这是一个非常重要的消息，因为它允许节点来互相发现彼此。我已经开始实现了，不过还没有完成！

链接：

1.  [Source codes](https://github.com/Jeiwan/blockchain_go/tree/part_7)
2.  [Bitcoin protocol documentation](https://en.bitcoin.it/wiki/Protocol_documentation)
3.  [Bitcoin network](https://en.bitcoin.it/wiki/Network)

原文：[Building Blockchain in Go. Part 7: Network](https://jeiwan.cc/posts/building-blockchain-in-go-part-7/)
