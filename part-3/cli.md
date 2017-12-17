持久化和命令行接口
==================

## CLI

到目前为止，我们的实现还没有提供一个与程序交互的接口：目前只是在 `main` 函数中简单执行了 `NewBlockchain` 和 `bc.AddBlock` 。是时候改变了！现在我们想要拥有这些命令：

```go
blockchain_go addblock "Pay 0.031337 for a coffee"
blockchain_go printchain
```

所有命令行相关的操作都会通过 `CLI` 结构进行处理：

```go
type CLI struct {
	bc *Blockchain
}
```

它的 “入口” 是 `Run` 函数：

```go
func (cli *CLI) Run() {
	cli.validateArgs()

	addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
	printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)

	addBlockData := addBlockCmd.String("data", "", "Block data")

	switch os.Args[1] {
	case "addblock":
		err := addBlockCmd.Parse(os.Args[2:])
	case "printchain":
		err := printChainCmd.Parse(os.Args[2:])
	default:
		cli.printUsage()
		os.Exit(1)
	}

	if addBlockCmd.Parsed() {
		if *addBlockData == "" {
			addBlockCmd.Usage()
			os.Exit(1)
		}
		cli.addBlock(*addBlockData)
	}

	if printChainCmd.Parsed() {
		cli.printChain()
	}
}
```

我们会使用标准库里面的 [flag](https://golang.org/pkg/flag/) 包来解析命令行参数：

```go
addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)
addBlockData := addBlockCmd.String("data", "", "Block data")
```

首先，我们创建两个子命令: `addblock` 和 `printchain`, 然后给 `addblock` 添加 `-data` 标志。`printchain` 没有任何标志。

```go
switch os.Args[1] {
case "addblock":
	err := addBlockCmd.Parse(os.Args[2:])
case "printchain":
	err := printChainCmd.Parse(os.Args[2:])
default:
	cli.printUsage()
	os.Exit(1)
}
```

然后，我们检查用户提供的命令，解析相关的 `flag` 子命令：

```go
if addBlockCmd.Parsed() {
	if *addBlockData == "" {
		addBlockCmd.Usage()
		os.Exit(1)
	}
	cli.addBlock(*addBlockData)
}

if printChainCmd.Parsed() {
	cli.printChain()
}
```

接着检查解析是哪一个子命令，并调用相关函数：

```go
func (cli *CLI) addBlock(data string) {
	cli.bc.AddBlock(data)
	fmt.Println("Success!")
}

func (cli *CLI) printChain() {
	bci := cli.bc.Iterator()

	for {
		block := bci.Next()

		fmt.Printf("Prev. hash: %x\n", block.PrevBlockHash)
		fmt.Printf("Data: %s\n", block.Data)
		fmt.Printf("Hash: %x\n", block.Hash)
		pow := NewProofOfWork(block)
		fmt.Printf("PoW: %s\n", strconv.FormatBool(pow.Validate()))
		fmt.Println()

		if len(block.PrevBlockHash) == 0 {
			break
		}
	}
}
```

这部分内容跟之前的很像，唯一的区别是我们现在使用的是 `BlockchainIterator` 对区块链中的区块进行迭代：

记得不要忘了对 `main` 函数作出相应的修改：

```go
func main() {
	bc := NewBlockchain()
	defer bc.db.Close()

	cli := CLI{bc}
	cli.Run()
}
```

注意，无论提供什么命令行参数，都会创建一个新的链。

这就是今天的所有内容了! 来看一下是不是如期工作：

```go
$ blockchain_go printchain
No existing blockchain found. Creating a new one...
Mining the block containing "Genesis Block"
000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true

$ blockchain_go addblock -data "Send 1 BTC to Ivan"
Mining the block containing "Send 1 BTC to Ivan"
000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13

Success!

$ blockchain_go addblock -data "Pay 0.31337 BTC for a coffee"
Mining the block containing "Pay 0.31337 BTC for a coffee"
000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148

Success!

$ blockchain_go printchain
Prev. hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
Data: Pay 0.31337 BTC for a coffee
Hash: 000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148
PoW: true

Prev. hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
Data: Send 1 BTC to Ivan
Hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
PoW: true

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true
```

![test](http://upload-images.jianshu.io/upload_images/127313-996c857601ed80a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


参考：

- [Full source codes](https://github.com/Jeiwan/blockchain_go/tree/part_3)

- [Bitcoin Core Data Storage](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage)

- [boltdb](https://github.com/boltdb/bolt)

- [encoding/gob](https://golang.org/pkg/encoding/gob/)

- [flag](https://golang.org/pkg/flag/)

- [part_3](https://github.com/Jeiwan/blockchain_go/tree/part_3)

- [Building Blockchain in Go. Part 3: Persistence and CLI](https://jeiwan.cc/posts/building-blockchain-in-go-part-3/)

----

- 上一节: [持久化](persistence.md)
