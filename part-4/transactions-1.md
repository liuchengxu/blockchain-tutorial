交易（1）
========

## 引言

交易（transaction）是比特币的核心所在，而区块链唯一的目的，也正是为了能够安全可靠地存储交易。在区块链中，交易一旦被创建，就没有任何人能够再去修改或是删除它。今天，我们将会开始实现交易。不过，由于交易是很大的话题，我会把它分为两部分来讲：在今天这个部分，我们会实现交易的基本框架。在第二部分，我们会继续讨论它的一些细节。

由于比特币采用的是 UTXO 模型，并非账户模型，并不直接存在“余额”这个概念，余额需要通过遍历整个交易历史得来。

## 比特币交易

点击 [这里](https://blockchain.info/zh-cn/tx/b6f6b339b546a13822192b06ccbdd817afea5311845f769702ae2912f7d94ab5) 在 blockchain.info 查看下图中的交易信息。

![tx](./img/blockchain-info-tx.png)

一笔交易由一些输入（input）和输出（output）组合而来：

```go
type Transaction struct {
	ID   []byte
	Vin  []TXInput
	Vout []TXOutput
}
```

对于每一笔新的交易，它的输入会引用（reference）之前一笔交易的输出（这里有个例外，coinbase 交易），引用就是花费的意思。所谓引用之前的一个输出，也就是将之前的一个输出包含在另一笔交易的输入当中，就是花费之前的交易输出。交易的输出，就是币实际存储的地方。下面的图示阐释了交易之间的互相关联：

![the interconnection of transactions](http://upload-images.jianshu.io/upload_images/127313-4f8e668c826fd31a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：

1. 有一些输出并没有被关联到某个输入上
2. 一笔交易的输入可以引用之前多笔交易的输出
3. 一个输入必须引用一个输出

贯穿本文，我们将会使用像“钱（money）”，“币（coin）”，“花费（spend）”，“发送（send）”，“账户（account）” 等等这样的词。但是在比特币中，其实并不存在这样的概念。交易仅仅是通过一个脚本（script）来锁定（lock）一些值（value），而这些值只可以被锁定它们的人解锁（unlock）。

----

- 下一节: [交易输出](txout.md)
