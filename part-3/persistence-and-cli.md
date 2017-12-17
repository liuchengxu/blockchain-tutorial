持久化和命令行接口
==================

## 引言

到目前为止，我们已经构建了一个有工作量证明机制的区块链。有了工作量证明，挖矿也就有了着落。虽然目前距离一个有着完整功能的区块链越来越近了，但是它仍然缺少了一些重要的特性。在今天的内容中，我们会将区块链持久化到一个数据库中，然后会提供一个简单的命令行接口，用来完成一些与区块链的交互操作。本质上，区块链是一个分布式数据库，不过，我们暂时先忽略 “分布式” 这个部分，仅专注于 “存储” 这一点。

## 选择数据库

目前，我们的区块链实现里面并没有用到数据库，而是在每次运行程序时，简单地将区块链存储在内存中。那么一旦程序退出，所有的内容就都消失了。我们没有办法再次使用这条链，也没有办法与其他人共享，所以我们需要把它存储到磁盘上。

那么，我们要用哪个数据库呢？实际上，任何一个数据库都可以。在 [比特币原始论文](https://bitcoin.org/bitcoin.pdf) 中，并没有提到要使用哪一个具体的数据库，它完全取决于开发者如何选择。 [Bitcoin Core](https://github.com/bitcoin/bitcoin) ，最初由中本聪发布，现在是比特币的一个参考实现，它使用的是  [LevelDB](https://github.com/google/leveldb)。而我们将要使用的是...

## BoltDB

因为它：

1. 非常简洁
2. 用 Go 实现
3. 不需要运行一个服务器
4. 能够允许我们构造想要的数据结构

BoltDB GitHub 上的 [README](https://github.com/boltdb/bolt) 是这么说的：

>Bolt 是一个纯键值存储的 Go 数据库，启发自 Howard Chu 的 LMDB. 它旨在为那些无须一个像 Postgres 和 MySQL 这样有着完整数据库服务器的项目，提供一个简单，快速和可靠的数据库。
>
>由于 Bolt 意在用于提供一些底层功能，简洁便成为其关键所在。它的 API 并不多，并且仅关注值的获取和设置。仅此而已。

听起来跟我们的需求完美契合！来快速过一下：

Bolt 使用键值存储，这意味着它没有像 SQL RDBMS （MySQL，PostgreSQL 等等）的表，没有行和列。相反，数据被存储为键值对（key-value pair，就像 Golang 的 map）。键值对被存储在 bucket 中，这是为了将相似的键值对进行分组（类似 RDBMS 中的表格）。因此，为了获取一个值，你需要知道一个 bucket 和一个键（key）。

需要注意的一个事情是，Bolt 数据库没有数据类型：键和值都是字节数组（byte array）。鉴于需要在里面存储 Go 的结构（准确来说，也就是存储**Block（块）**），我们需要对它们进行序列化，也就说，实现一个从 Go struct 转换到一个 byte array 的机制，同时还可以从一个 byte array 再转换回 Go struct。虽然我们将会使用  [encoding/gob](https://golang.org/pkg/encoding/gob/)  来完成这一目标，但实际上也可以选择使用 **JSON**, **XML**, **Protocol Buffers** 等等。之所以选择使用 **encoding/gob**, 是因为它很简单，而且是 Go 标准库的一部分。

虽然 BoldDB 的作者出于个人原因已经不在对其维护（见[README](https://github.com/boltdb/bolt/commit/fa5367d20c994db73282594be0146ab221657943)）, 不过关系不大，它已经足够稳定了，况且也有活跃的 fork:[coreos/bblot](https://github.com/coreos/bbolt)。

## 数据库结构

在开始实现持久化的逻辑之前，我们首先需要决定到底要如何在数据库中进行存储。为此，我们可以参考 Bitcoin Core 的做法：

简单来说，Bitcoin Core 使用两个 “bucket” 来存储数据：

1. 其中一个 bucket 是 **blocks**，它存储了描述一条链中所有块的元数据
2. 另一个 bucket 是 **chainstate**，存储了一条链的状态，也就是当前所有的未花费的交易输出，和一些元数据

此外，出于性能的考虑，Bitcoin Core 将每个区块（block）存储为磁盘上的不同文件。如此一来，就不需要仅仅为了读取一个单一的块而将所有（或者部分）的块都加载到内存中。但是，为了简单起见，我们并不会实现这一点。

在 **blocks** 中，**key -> value** 为：

key                                                | value
:----:                                             | :----:
`b` + 32 字节的 block hash                         | block index record
`f` + 4 字节的 file number                         | file information record
`l` + 4 字节的 file number                         | the last block file number used
`R` + 1 字节的 boolean                             | 是否正在 reindex
`F` + 1 字节的 flag name length + flag name string | 1 byte boolean: various flags that can be on or off
`t` + 32 字节的 transaction hash                   | transaction index record

在 **chainstate**，**key -> value** 为：

key                              | value
:----:                           | :----:
`c` + 32 字节的 transaction hash | unspent transaction output record for that transaction
`B`                              | 32 字节的 block hash: the block hash up to which the database represents the unspent transaction outputs

详情可见 **[这里](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage)**。

因为目前还没有交易，所以我们只需要 **blocks** bucket。另外，正如上面提到的，我们会将整个数据库存储为单个文件，而不是将区块存储在不同的文件中。所以，我们也不会需要文件编号（file number）相关的东西。最终，我们会用到的键值对有：

1. 32 字节的 block-hash -> block 结构
2. `l` -> 链中最后一个块的 hash

这就是实现持久化机制所有需要了解的内容了。

## 序列化

上面提到，在 BoltDB 中，值只能是 `[]byte` 类型，但是我们想要存储 `Block` 结构。所以，我们需要使用 [encoding/gob](https://golang.org/pkg/encoding/gob/) 来对这些结构进行序列化。

让我们来实现 `Block` 的 `Serialize` 方法（为了简洁起见，此处略去了错误处理）：

```go
func (b *Block) Serialize() []byte {
	var result bytes.Buffer
	encoder := gob.NewEncoder(&result)

	err := encoder.Encode(b)

	return result.Bytes()
}
```

这个部分比较直观：首先，我们定义一个 buffer 存储序列化之后的数据。然后，我们初始化一个 gob `encoder` 并对 block 进行编码，结果作为一个字节数组返回。

接下来，我们需要一个解序列化的函数，它会接受一个字节数组作为输入，并返回一个 `Block`. 它不是一个方法（method），而是一个单独的函数（function）：

```go
func DeserializeBlock(d []byte) *Block {
	var block Block

	decoder := gob.NewDecoder(bytes.NewReader(d))
	err := decoder.Decode(&block)

	return &block
}
```

这就是序列化部分的内容了。

----

- 下一节: [持久化](persistence.md)
