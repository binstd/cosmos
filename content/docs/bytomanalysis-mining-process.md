---
id: bytomanalysis-mining-process
title: 挖矿流程
permalink: docs/bytomanalysis-mining-process.html
layout: docs
---

当我们以`bytom init --chain_id=solonet`建立比原单机节点用于本地测试时，很快会发现自己将面临一个尴尬的问题：余额为0。就算我们使用`bytom node --mining`开启挖矿，理论上由于我们是单机状态，本机算力就是全网算力，应该每次都能够挖到，但是不知道为什么，在我尝试的时候发现总是挖不到，所以打算简单研究一下比原的挖矿流程，看看有没有办法能改点什么，给自己单机多挖点BTM以方便后面的测试。

所以在今天我打算通过源代码分析一下比原的挖矿流程，但是考虑到它肯定会涉及到比原的核心，所以太复杂的地方我就会先跳过，那些地方时机成熟的时候会彻底研究一下。

如果我们快速搜索一下，就能发现在比原代码中有一个类型叫`CPUMiner`，我们围绕着它应该就可以了。

首先还是从比原启动开始，看看`CPUMiner`是如何被启动的。

下面是`bytom node --mining`对应的入口函数：

[cmd/bytomd/main.go#L54-L57](https://github.com/freewind/bytom-v1.0.1/blob/master/cmd/bytomd/main.go#L54-L57)

```go
func main() {
    cmd := cli.PrepareBaseCmd(commands.RootCmd, "TM", os.ExpandEnv(config.DefaultDataDir()))
    cmd.Execute()
}
```

由于传入了参数`node`，所以创建Node并启动：

[cmd/bytomd/commands/run_node.go#L41-L54](https://github.com/freewind/bytom-v1.0.1/blob/master/cmd/bytomd/commands/run_node.go#L41-L54)

```go
func runNode(cmd *cobra.Command, args []string) error {
    // Create & start node
    n := node.NewNode(config)
    if _, err := n.Start(); err != nil {
        // ...
}
```

在创建一个Node对象的时候，也会创建`CPUMiner`对象：

[node/node.go#L59-L142](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L59-L142)

```go
func NewNode(config *cfg.Config) *Node {
    // ...
    node.cpuMiner = cpuminer.NewCPUMiner(chain, accounts, txPool, newBlockCh)
    node.miningPool = miningpool.NewMiningPool(chain, accounts, txPool, newBlockCh)
    // ...
    return node
}
```

这里可以看到创建了两个与挖矿相关的东西，一个是`NewCPUMiner`，另一个是`miningPool`。我们先看`NewCPUMiner`对应的代码：

[mining/cpuminer/cpuminer.go#L282-L293](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/cpuminer/cpuminer.go#L282-L293)

```go
func NewCPUMiner(c *protocol.Chain, accountManager *account.Manager, txPool *protocol.TxPool, newBlockCh chan *bc.Hash) *CPUMiner {
    return &CPUMiner{
        chain:             c,
        accountManager:    accountManager,
        txPool:            txPool,
        numWorkers:        defaultNumWorkers,
        updateNumWorkers:  make(chan struct{}),
        queryHashesPerSec: make(chan float64),
        updateHashes:      make(chan uint64),
        newBlockCh:        newBlockCh,
    }
}
```

从这里的字段可以看到，CPUMiner在工作的时候：

- 可能需要用到外部的三个对象分别是：`chain`（代表本机持有的区块链），`accountManager`（管理帐户），`txPool`（交易池）
- `numWorkers`：应该保持几个worker在挖矿，默认值`defaultNumWorkers`为常量`1`，也就是说默认只有一个worker。这对于多核cpu来说有点亏，真要挖矿的话可以把它改大点，跟核心数相同（不过用普通电脑不太可能挖到了）
- `updateNumWorkers`：外界如果想改变worker的数量，可以通过向这个通道发消息实现。CPUMiner会监听它，并按要求增减worker
- `queryHashesPerSec`：这个没用上，忽略吧。我发现比原的开发人员很喜欢预先设计，有很多这样没用上的代码
- `updateHashes`: 这个没用上，忽略
- `newBlockCh`: 一个来自外部的通道，用来告诉外面自己成功挖到了块，并且已经放进了本地区块链，其它地方就可以用它了（比如广播出去）

然而这里出现的并不是`CPUMiner`全部的字段，仅仅是需要特意初始化的几个。完整的在这里:

[mining/cpuminer/cpuminer.go#L29-L45](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/cpuminer/cpuminer.go#L29-L45)

```go
type CPUMiner struct {
    sync.Mutex
    chain             *protocol.Chain
    accountManager    *account.Manager
    txPool            *protocol.TxPool
    numWorkers        uint64
    started           bool
    discreteMining    bool
    wg                sync.WaitGroup
    workerWg          sync.WaitGroup
    updateNumWorkers  chan struct{}
    queryHashesPerSec chan float64
    updateHashes      chan uint64
    speedMonitorQuit  chan struct{}
    quit              chan struct{}
    newBlockCh        chan *bc.Hash
}
```

可以看到还多出了几个：

- `sync.Mutex`：为CPUMiner提供了锁，方便在不同的goroutine代码中进行同步
- `started`：记录miner是否启动了
- `discreteMining`：这个在当前代码中没有赋过值，永远是`false`，我觉得应该删除。已提[issue #961](https://github.com/Bytom/bytom/issues/961)
- `wg`和`workerWg`：都是跟控制goroutine流程相关的
- `speedMonitorQuit`：也没什么用，忽略
- `quit`：外界可以给这个通道发消息来通知CPUMiner退出

再回到`n.Start`看看`cpuMiner`是何时启动的：

[node/node.go#L169-L180](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L169-L180)

```go
func (n *Node) OnStart() error {
    if n.miningEnable {
        n.cpuMiner.Start()
    }
    // ...
}
```

由于我们传入了参数`--mining`，所以`n.miningEnable`是`true`，于是`n.cpuMiner.Start`会运行：

[mining/cpuminer/cpuminer.go#L188-L205](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/cpuminer/cpuminer.go#L188-L205)

```go
func (m *CPUMiner) Start() {
    m.Lock()
    defer m.Unlock()

    if m.started || m.discreteMining {
        return
    }

    m.quit = make(chan struct{})
    m.speedMonitorQuit = make(chan struct{})
    m.wg.Add(1)
    go m.miningWorkerController()

    m.started = true
    log.Infof("CPU miner started")
}
```

这段代码没太多需要说的，主要是通过判断`m.started`保证不会重复启动，然后把真正的工作放在了`m.miningWorkerController()`中：

[mining/cpuminer/cpuminer.go#L126-L125](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/cpuminer/cpuminer.go#L126-L125)

```go
func (m *CPUMiner) miningWorkerController() {
    // 1. 
    var runningWorkers []chan struct{}
    launchWorkers := func(numWorkers uint64) {
        for i := uint64(0); i < numWorkers; i++ {
            quit := make(chan struct{})
            runningWorkers = append(runningWorkers, quit)

            m.workerWg.Add(1)
            go m.generateBlocks(quit)
        }
    }
    runningWorkers = make([]chan struct{}, 0, m.numWorkers)
    launchWorkers(m.numWorkers)

out:
    for {
        select {
        // 2. 
        case <-m.updateNumWorkers:
            numRunning := uint64(len(runningWorkers))
            if m.numWorkers == numRunning {
                continue
            }

            if m.numWorkers > numRunning {
                launchWorkers(m.numWorkers - numRunning)
                continue
            }

            for i := numRunning - 1; i >= m.numWorkers; i-- {
                close(runningWorkers[i])
                runningWorkers[i] = nil
                runningWorkers = runningWorkers[:i]
            }

        // 3.
        case <-m.quit:
            for _, quit := range runningWorkers {
                close(quit)
            }
            break out
        }
    }

    m.workerWg.Wait()
    close(m.speedMonitorQuit)
    m.wg.Done()
}
```

这个方法看起来代码挺多的，但是实际上做的事情还是比较好理清的，主要是做了三件事：

1. 第1处代码是按指定的worker数量启动挖矿例程
2. 第2处是监听应该保持的worker数量并增减
3. 第3处在被知关闭的时候安全关闭

代码比较清楚，应该不需要多讲。

可以看第1处代码中，真正挖矿的工作是放在`generateBlocks`里的：

[mining/cpuminer/cpuminer.go#L84-L119](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/cpuminer/cpuminer.go#L84-L119)

```go
func (m *CPUMiner) generateBlocks(quit chan struct{}) {
    ticker := time.NewTicker(time.Second * hashUpdateSecs)
    defer ticker.Stop()

out:
    for {
        select {
        case <-quit:
            break out
        default:
        }

        // 1.
        block, err := mining.NewBlockTemplate(m.chain, m.txPool, m.accountManager)
        // ...

        // 2.
        if m.solveBlock(block, ticker, quit) {
            // 3.
            if isOrphan, err := m.chain.ProcessBlock(block); err == nil {
                // ...
                // 4.
                blockHash := block.Hash()
                m.newBlockCh <- &blockHash
                // ...
            }
        }
    }

    m.workerWg.Done()
}
```

方法里省略了一些不太重要的代码，我们可以从标注的几处看一下在做什么：

1. 第1处通过`mining.NewBlockTemplate`根据模板生成了一个block
2. 第2处是以暴力方式（从`0`开始挨个计算）来争夺对该区块的记帐权
3. 第3处是通过`chain.ProcessBlock(block)`尝试把它加到本机持有的区块链上
4. 第4处是向`newBlockCh`通道发出消息，通知外界自己挖到了新的块

### mining.NewBlockTemplate

我们先看一下第1处中的`mining.NewBlockTemplate`：

[mining/mining.go#L67-L154](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/mining.go#L67-L154)

```go
func NewBlockTemplate(c *protocol.Chain, txPool *protocol.TxPool, accountManager *account.Manager) (b *types.Block, err error) {
    // ...
    return b, err
}
```

这个方法很长，但是内容都被我忽略了，原因是它的内容过于细节，并且已经触及到了比原的核心，所以现在大概了解一下就可以了。

比原在一个Block区块里，有一些基本信息，比如在其头部有前一块的hash值、挖矿难度值、时间戳等等，主体部有各种交易记录，以及多次层的hash摘要。在这个方法中，主要的逻辑就是去找到这些信息然后把它们包装成一个Block对象，然后交由后面处理。我觉得在我们还没有深刻理解比原的区块链结构和规则的情况下，看这些太细节的东西没有太大用处，所以先忽略，等以后合适的时候再回过头来看就简单了。

### m.solveBlock

我们继续向下，当由`NewBlockTemplate`生成好了一个Block对象后，它会交给`solveBlock`方法处理：

[mining/cpuminer/cpuminer.go#L50-L75](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/cpuminer/cpuminer.go#L50-L75)

```go
func (m *CPUMiner) solveBlock(block *types.Block, ticker *time.Ticker, quit chan struct{}) bool {
    // 1. 
    header := &block.BlockHeader
    seed, err := m.chain.CalcNextSeed(&header.PreviousBlockHash)
    // ...

    // 2.
    for i := uint64(0); i <= maxNonce; i++ {
        // 3. 
        select {
        case <-quit:
            return false
        case <-ticker.C:
            if m.chain.BestBlockHeight() >= header.Height {
                return false
            }
        default:
        }

        // 4.
        header.Nonce = i
        headerHash := header.Hash()
        
        // 5.
        if difficulty.CheckProofOfWork(&headerHash, seed, header.Bits) {
            return true
        }
    }
    return false
}
```

这个方法就是挖矿中我们最关心的部分了：争夺记帐权。

我把代码分成了4块，依次简单讲解：

1. 第1处是从本地区块链中找到新生成的区块指定的父区块，并由它计算出来`seed`，它是如何计算出来的我们暂时不关心（比较复杂），此时只要知道它是用来检查工作量的就可以了
2. 第2处是使用暴力方式来计算目标值，用于争夺记帐权。为什么说是暴力方式？因为挖矿的算法保证了想解开难题，没有比从0开始一个个计算更快的办法，所以这里从0开始依次尝试，直到`maxNonce`结束。`maxNonce`是一个非常大的数`^uint64(0)`（即`2^64 - 1`），基本上是不可能在一个区块时间内遍历完的。
3. 第3处是在每次循环中进行计算之前，都看一看是否需要退出。在两种情况下应该退出，一是`quit`通道里有新消息，被人提醒退出（可能是时间到了）；另一种是本地的区块链中已经收到了新的块，且高度比较自己高，说明已经有别人抢到了。
4. 第4处是把当前循环的数字当作`Nonce`，计算出Hash值
5. 第5处是调用`difficulty.CheckProofOfWork`来检查当前算出来的hash值是否满足了当前难度。如果满足就说明自己拥有了记帐权，这个块是有效的；否则就继续计算

然后我们再看一下第5处的`difficulty.CheckProofOfWork`:

[consensus/difficulty/difficulty.go#L120-L123](https://github.com/freewind/bytom-v1.0.1/blob/master/consensus/difficulty/difficulty.go#L120-L123)

```go
func CheckProofOfWork(hash, seed *bc.Hash, bits uint64) bool {
    compareHash := tensority.AIHash.Hash(hash, seed)
    return HashToBig(compareHash).Cmp(CompactToBig(bits)) <= 0
}
```

在这个方法里，可以看到出现了一个`tensority.AIHash`，这是比原独有的人工智能友好的工作量算法，相关论文的下载地址：<https://github.com/Bytom/bytom/wiki/download/tensority-v1.2.pdf>，有兴趣的同学可以去看看。由于这个算法的难度肯定超出了本文的预期，所以就不研究它了。在以后，如果有机会有条件的话，也许我会试着理解一下（不要期待~）

从这个方法里可以看出，它是调用了`tensority.AIHash`中的相关方法进判断当前计算出来的hash是否满足难度要求。

在本文的开始，我们说过希望能找到一种方法修改比原的代码，让我们在`solonet`模式下，可以正常挖矿，得到BTM用于测试。看到这个方法的时候，我觉得已经找到了，我们只需要修改一下让它永远返回`true`即可：

```go
func CheckProofOfWork(hash, seed *bc.Hash, bits uint64) bool {
    compareHash := tensority.AIHash.Hash(hash, seed)
    return HashToBig(compareHash).Cmp(CompactToBig(bits)) <= 0 || true
}
```

这里也许会让人觉得有点奇怪，为什么要在最后的地方加上`|| true`，而不是在前面直接返回`true`呢？这是因为，如果直接返回`true`，可能使得程序中关于时间戳检查的地方出现问题，出现如下的错误：

```
time="2018-05-17T12:10:14+08:00" level=error msg="Miner fail on ProcessBlock block, timestamp is not in the valid range: invalid block" height=32
```

原因还未深究，可能是因为原本的代码是需要消耗一些时间的，正好使得检查通过。如果直接返回`true`就太快了，反而使检查通过不了。不过我感觉这里是有一点问题的，留待以后再研究。

这样修改完以后，再重新编译并启动比原节点，每个块都能挖到了，差不多一秒一个块（一下子变成大富豪了：）

### m.chain.ProcessBlock

我们此时该回到`generateBlocks`方法中的第3处，即：

[mining/cpuminer/cpuminer.go#L84-L119](https://github.com/freewind/bytom-v1.0.1/blob/master/mining/cpuminer/cpuminer.go#L84-L119)

```go
func (m *CPUMiner) generateBlocks(quit chan struct{}) {
        //...
        if m.solveBlock(block, ticker, quit) {
            // 3.
            if isOrphan, err := m.chain.ProcessBlock(block); err == nil {
                // ...
                // 4.
                blockHash := block.Hash()
                m.newBlockCh <- &blockHash
                // ...
            }
        }
    }

    m.workerWg.Done()
}
```

`m.chain.ProcessBlock`把刚才成功拿到记帐权的块向本地区块链上添加：

[protocol/block.go#L191-L196](https://github.com/freewind/bytom-v1.0.1/blob/master/protocol/block.go#L191-L196)

```go
func (c *Chain) ProcessBlock(block *types.Block) (bool, error) {
    reply := make(chan processBlockResponse, 1)
    c.processBlockCh <- &processBlockMsg{block: block, reply: reply}
    response := <-reply
    return response.isOrphan, response.err
}
```

可以看到这里实际上是把这个工作甩出去了，因为它把要处理的块放进了`Chain.processBlockCh`这个通道里，同时传过去的还有一个用于对方回复的通道`reply`。然后监听`reply`等消息就可以了。

那么谁将会处理`c.processBlockCh`里的内容呢？当然是由`Chain`，只不过这里就属于比原核心了，我们留等以后再详细研究，今天就先跳过。

如果处理完没有出错，就进入到了第4块，把这个block的hash放在`newBlockCh`通道里。这个`newBlockCh`是由外面传入的，很多地方都会用到。当它里面有新的数据时，就说明本机挖到了新块（并且已经添加到了本机的区块链上），其它的地方就可以使用它进行别的操作（比如广播出去）

那么到这里，我们今天的问题就算解决了，留下了很多坑，以后专门填。

---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！