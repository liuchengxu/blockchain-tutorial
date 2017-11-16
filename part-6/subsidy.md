交易（2）
========

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


