---
id: bytomanalysis-howto-getdata
title: 从比原节点拿到区块数据
permalink: docs/bytomanalysis-howto-getdata.html
layout: docs
---

在前一篇中，我们已经知道如何连上一个比原节点的p2p端口，并与对方完成身份验证。此时，双方结点已经建立起来了信任，并且连接也不会断开，下一步，两者就可以继续交换数据了。

那么，我首先想到的就是，如何才能让对方把它已有的区块数据全都发给我呢？

这其实可以分为三个问题：

1. 我需要发给它什么样的数据？
2. 它在内部由是如何应答的呢？
3. 我拿到数据之后，应该怎么处理？

由于这一块的逻辑还是比较复杂的，所以在本篇我们先回答第一个问题：

我们要发送什么样的数据请求，才能让比原节点把它持有的区块数据发给我？
-------------------------------------------------------

### 找到发送请求的代码

首先我们先要在代码中定位到，比原到底是在什么时候来向对方节点发送请求的。

在前一篇讲的是如何建立连接并验证身份，那么发出数据请求的操作，一定在上次的代码之后。按照这个思路，我们在`SyncManager`类中`Switch`启动之后，找到了一个叫`BlockKeeper`的类，相关的操作是在它里面完成的。

下面是老规矩，还是从启动开始，但是会更简化一些：

[cmd/bytomd/main.go#L54](https://github.com/freewind/bytom-v1.0.1/blob/master/cmd/bytomd/main.go#L54)
```go
func main() {
    cmd := cli.PrepareBaseCmd(commands.RootCmd, "TM", os.ExpandEnv(config.DefaultDataDir()))
    cmd.Execute()
}
```

[cmd/bytomd/commands/run_node.go#L41](https://github.com/freewind/bytom-v1.0.1/blob/master/cmd/bytomd/commands/run_node.go#L41)
```go
func runNode(cmd *cobra.Command, args []string) error {
    n := node.NewNode(config)
    if _, err := n.Start(); err != nil {
        // ...
}
```

[node/node.go#L169](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L169)
```go
func (n *Node) OnStart() error {
    // ...
    n.syncManager.Start()
    // ...
}
```

[netsync/handle.go#L141](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/handle.go#L141)

```go
func (sm *SyncManager) Start() {
    go sm.netStart()
    // ...
    go sm.syncer()
}
```

注意`sm.netStart()`，我们在一篇中建立连接并验证身份的操作，就是在它里面完成的。而这次的这个问题，是在下面的`sm.syncer()`中完成的。

另外注意，由于这两个函数调用都使用了goroutine，所以它们是同时进行的。

`sm.syncer()`的代码如下：

[netsync/sync.go#L46](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/sync.go#L46)

```go
func (sm *SyncManager) syncer() {
    sm.fetcher.Start()
    defer sm.fetcher.Stop()

    // ...
    for {
        select {
        case <-sm.newPeerCh:
            log.Info("New peer connected.")
            // Make sure we have peers to select from, then sync
            if sm.sw.Peers().Size() < minDesiredPeerCount {
                break
            }
            go sm.synchronise()
            // ..
    }
}
```

这里混入了一个叫`fetcher`的奇怪的东西，名字看起来好像是专门去抓取数据的，我们要找的是它吗？

可惜不是，`fetcher`的作用是从多个peer那里拿到了区块数据之后，对数据进行整理，把有用的放到本地链上。我们在以后会研究它，所以这里不展开讨论。

接着是一个`for`循环，当发现通道`newPeerCh`有了新数据（也就是有了新的节点连接上了），会判断一下当前自己连着的节点是否够多（大于等于`minDesiredPeerCount`，值为`5`），够多的话，就会进入`sm.synchronise()`，进行数据同步。

这里为什么要多等几个节点，而不是一连上就马上同步呢？我想这是希望有更多选择的机会，找到一个数据够多的节点。

`sm.synchronise()`还是属于`SyncManager`的方法。在真正调用到`BlockKeeper`的方法之前，它还做了一些比如清理已经断开的peer，找到最适合同步数据的peer等。其中“清理peer”的工作涉及到不同的对象持有的peer集合间的同步，略有些麻烦，但对当前问题帮助不大，所以我打算把它们放在以后的某个问题中回答（比如“当一个节点断开了，比原会有什么样的处理”），这里就先省略。

`sm.synchronise()`代码如下：

[netsync/sync.go#L77](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/sync.go#L77)
```go
func (sm *SyncManager) synchronise() {
    log.Info("bk peer num:", sm.blockKeeper.peers.Len(), " sw peer num:", sm.sw.Peers().Size(), " ", sm.sw.Peers().List())
    // ...
    peer, bestHeight := sm.peers.BestPeer()
    // ...
    if bestHeight > sm.chain.BestBlockHeight() {
        // ...
        sm.blockKeeper.BlockRequestWorker(peer.Key, bestHeight)
    }
}
```

可以看到，首先是从众多的peers中，找到最合适的那个。什么叫Best呢？看一下`BestPeer()`的定义：

[netsync/peer.go#L266](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/peer.go#L266)

```go
func (ps *peerSet) BestPeer() (*p2p.Peer, uint64) {
    // ...
    for _, p := range ps.peers {
        if bestPeer == nil || p.height > bestHeight {
            bestPeer, bestHeight = p.swPeer, p.height
        }
    }
    return bestPeer, bestHeight
}
```
其实就是持有区块链数据最长的那个。

找到了BestPeer之后，就调用`sm.blockKeeper.BlockRequestWorker(peer.Key, bestHeight)`方法，从这里，正式进入`BlockKeeper` －－ 也就是本文的主角 －－ 的世界。

### BlockKeeper

`blockKeeper.BlockRequestWorker`的逻辑比较复杂，它包含了：

1. 根据自己持有的区块数据来计算需要同步的数据
2. 向前面找到的最佳节点发送数据请求
3. 拿到对方发过来的区块数据
4. 对数据进行处理
5. 广播新状态
6. 处理各种出错情况，等等

由于本文中只关注“发送请求”，所以一些与之关系不大的逻辑我会忽略掉，留待以后再讲。

在“发送请求”这里，实际也包含了两种情形，一种简单的，一种复杂的：

1. 简单的：假设不存在分叉，则直接检查本地高度最高的区块，然后请求下一个区块
2. 复杂的：考虑分叉的情况，则当前本地的区块可能就存在分叉，那么到底应该请求哪个区块，就需要慎重考虑

由于第2种情况对于本文来说过于复杂（因为需要深刻理解比原链中分叉的处理逻辑），所以在本文中将把问题简化，只考虑第1种。而分叉的处理，将放在以后讲解。

下面是把`blockKeeper.BlockRequestWorker`中的代码简化成了只包含第1种情况：

[netsync/block_keeper.go#L72](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/block_keeper.go#L72)

```go
func (bk *blockKeeper) BlockRequestWorker(peerID string, maxPeerHeight uint64) error {
    num := bk.chain.BestBlockHeight() + 1
    reqNum := uint64(0)
    reqNum = num
    // ...
    bkPeer, ok := bk.peers.Peer(peerID)
    swPeer := bkPeer.getPeer()
    // ...
    block, err := bk.BlockRequest(peerID, reqNum)
    // ...
}
```

在这种情况下，我们可以认为`bk.chain.BestBlockHeight()`中的`Best`，指的是本地持有的不带分叉的区块链高度最高的那个。（需要提醒的是，如果存在分叉情况，则`Best`不一定是高度最高的那个）

那么我们就可以直接向最佳peer请求下一个高度的区块，它是通过`bk.BlockRequest(peerID, reqNum)`实现的：

[netsync/block_keeper.go#L152](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/block_keeper.go#L152)

```go
func (bk *blockKeeper) BlockRequest(peerID string, height uint64) (*types.Block, error) {
    var block *types.Block

    if err := bk.blockRequest(peerID, height); err != nil {
        return nil, errReqBlock
    }

    // ...

    for {
        select {
        case pendingResponse := <-bk.pendingProcessCh:
            block = pendingResponse.block
            // ...
            return block, nil
        // ...
        }
    }
}
```

在上面简化后的代码中，主要分成了两个部分。一个是发送请求`bk.blockRequest(peerID, height)`，这是本文的重点；它下面的`for-select`部分，已经是在等待并处理对方节点的返回数据了，这部分我们今天先略过不讲。

`bk.blockRequest(peerID, height)`这个方法，从逻辑上又可以分成两部分：

1. 构造出请求的信息
2. 把信息发送给对方节点

### 构造出请求的信息

`bk.blockRequest(peerID, height)`经过一连串的方法调用之后，使用`height`构造出了一个`BlockRequestMessage`对象，代码如下：

[netsync/block_keeper.go#L148](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/block_keeper.go#L148)

```go
func (bk *blockKeeper) blockRequest(peerID string, height uint64) error {
    return bk.peers.requestBlockByHeight(peerID, height)
}
```

[netsync/peer.go#L332](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/peer.go#L332)
```go
func (ps *peerSet) requestBlockByHeight(peerID string, height uint64) error {
    peer, ok := ps.Peer(peerID)
    // ...
    return peer.requestBlockByHeight(height)
}
```

[netsync/peer.go#L73](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/peer.go#L73)

```go
func (p *peer) requestBlockByHeight(height uint64) error {
    msg := &BlockRequestMessage{Height: height}
    p.swPeer.TrySend(BlockchainChannel, struct{ BlockchainMessage }{msg})
    return nil
}
```

到这里，终于构造出了所需要的`BlockRequestMessage`，其实主要就是把`height`告诉peer。

然后，通过`Peer`的`TrySend()`把该信息发出去。

### 发送请求

在`TrySend`中，主要是通过`github.com/tendermint/go-wire`库将其序列化，再发送给对方。看起来应该是很简单的操作吧，先预个警，还是挺绕的。

当我们进入`TrySend()`后：

[p2p/peer.go#L242](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/peer.go#L242)
```go
func (p *Peer) TrySend(chID byte, msg interface{}) bool {
    if !p.IsRunning() {
        return false
    }
    return p.mconn.TrySend(chID, msg)
}
```

发现它把锅丢给了`p.mconn.TrySend`方法，那么`mconn`是什么？`chID`又是什么？

`mconn`是`MConnection`的实例，它是从哪儿来的？它应该在之前的某个地方初始化了，否则我们没法直接调用它。所以我们先来找到它初始化的地方。

经过一番寻找，发现原来是在前一篇之后，即比原节点与另一个节点完成了身份验证之后，具体的位置在`Switch`类启动的地方。

我们这次直接从`Swtich`的`OnStart`作为起点：

[p2p/switch.go#L186](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L186)

```go
func (sw *Switch) OnStart() error {
    //...
    // Start listeners
    for _, listener := range sw.listeners {
        go sw.listenerRoutine(listener)
    }
    return nil
}
```

[p2p/switch.go#L498](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L498)

```go
func (sw *Switch) listenerRoutine(l Listener) {
    for {
        inConn, ok := <-l.Connections()
        // ...
        err := sw.addPeerWithConnectionAndConfig(inConn, sw.peerConfig)
        // ...
    }
}
```

[p2p/switch.go#L645](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L645)

```go
func (sw *Switch) addPeerWithConnectionAndConfig(conn net.Conn, config *PeerConfig) error {
    // ...
    peer, err := newInboundPeerWithConfig(conn, sw.reactorsByCh, sw.chDescs, sw.StopPeerForError, sw.nodePrivKey, config)
    // ...
}
```

[p2p/peer.go#L87](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/peer.go#L87)

```go
func newInboundPeerWithConfig(conn net.Conn, reactorsByCh map[byte]Reactor, chDescs []*ChannelDescriptor, onPeerError func(*Peer, interface{}), ourNodePrivKey crypto.PrivKeyEd25519, config *PeerConfig) (*Peer, error) {
    return newPeerFromConnAndConfig(conn, false, reactorsByCh, chDescs, onPeerError, ourNodePrivKey, config)
}
```

[p2p/peer.go#L91](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/peer.go#L91)

```go
func newPeerFromConnAndConfig(rawConn net.Conn, outbound bool, reactorsByCh map[byte]Reactor, chDescs []*ChannelDescriptor, onPeerError func(*Peer, interface{}), ourNodePrivKey crypto.PrivKeyEd25519, config *PeerConfig) (*Peer, error) {
    conn := rawConn
    // ...
    if config.AuthEnc {
        // ...
        conn, err = MakeSecretConnection(conn, ourNodePrivKey)
        // ...
    }

    // Key and NodeInfo are set after Handshake
    p := &Peer{
        outbound: outbound,
        conn:     conn,
        config:   config,
        Data:     cmn.NewCMap(),
    }

    p.mconn = createMConnection(conn, p, reactorsByCh, chDescs, onPeerError, config.MConfig)

    p.BaseService = *cmn.NewBaseService(nil, "Peer", p)

    return p, nil
}
```

终于找到了。上面方法中的`MakeSecretConnection`就是与对方节点交换公钥并进行身份验证的地方，下面的`p.mconn = createMConnection(...)`就是创建`mconn`的地方。

继续进去：

[p2p/peer.go#L292](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/peer.go#L292)

```go
func createMConnection(conn net.Conn, p *Peer, reactorsByCh map[byte]Reactor, chDescs []*ChannelDescriptor, onPeerError func(*Peer, interface{}), config *MConnConfig) *MConnection {
    onReceive := func(chID byte, msgBytes []byte) {
        reactor := reactorsByCh[chID]
        if reactor == nil {
            if chID == PexChannel {
                return
            } else {
                cmn.PanicSanity(cmn.Fmt("Unknown channel %X", chID))
            }
        }
        reactor.Receive(chID, p, msgBytes)
    }

    onError := func(r interface{}) {
        onPeerError(p, r)
    }

    return NewMConnectionWithConfig(conn, chDescs, onReceive, onError, config)
}
```

原来`mconn`是`MConnection`的实例，它是通过`NewMConnectionWithConfig`创建的。

看了上面的代码，发现这个`MConnectionWithConfig`与普通的`net.Conn`并没有太大的区别，只不过是当收到了对方发来的数据后，会根据指定的`chID`调用相应的`Reactor`的`Receive`方法来处理。所以它起到了将数据分发给`Reactor`的作用。

为什么需要这样的分发操作呢？这是因为，在比原中，节点之间交换数据，有多种不同的方式：

1. 一种是规定了详细的数据交互协议（比如有哪些信息类型，分别代表什么意思，什么情况下发哪个，如何应答等），在`ProtocolReactor`中实现，它对应的`chID`是`BlockchainChannel`，值为`byte(0x40)`
2. 另一种使用了与BitTorrent类似的文件共享协议，叫[PEX](https://en.wikipedia.org/wiki/Peer_exchange)，在`PEXReactor`中实现，它对应的`chID`是`PexChannel`，值为`byte(0x00)`

所以节点之间发送信息的时候，需要知道对方发过来的数据对应的是哪一种方式，然后转交给相应的`Reactor`去处理。

在比原中，前者是主要的方式，后者起到辅助作用。我们目前的文章中涉及到的都是前者，后者将在以后专门研究。

### `p.mconn.TrySend`

当我们知道了`p.mconn.TrySend`中的`mconn`是什么，并且在什么时候初始化以后，下面就可以进入它的`TrySend`方法了。

[p2p/connection.go#L243](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L243)

```go
func (c *MConnection) TrySend(chID byte, msg interface{}) bool {
    // ...
    channel, ok := c.channelsIdx[chID]
    // ...
    ok = channel.trySendBytes(wire.BinaryBytes(msg))
    if ok {
        // Wake up sendRoutine if necessary
        select {
        case c.send <- struct{}{}:
        default:
        }
    }

    return ok
}
```

可以看到，它找到相应的channel后（在这里应该是`ProtocolReactor`对应的channel），调用channel的`trySendBytes`方法。在发送数据的时候，使用了`github.com/tendermint/go-wire`库，将`msg`序列化为二进制数组。


[p2p/connection.go#L602](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L602)
```go
func (ch *Channel) trySendBytes(bytes []byte) bool {
    select {
    case ch.sendQueue <- bytes:
        atomic.AddInt32(&ch.sendQueueSize, 1)
        return true
    default:
        return false
    }
}
```

原来它是把要发送的数据，放到了该channel对应的`sendQueue`中，交由别人来发送。具体是由谁来发送，我们马上要就找到它。

细心的同学会发现，`Channel`除了`trySendBytes`方法外，还有一个`sendBytes`（在本文中没有用上）：

[p2p/connection.go#L589](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L589)
```go
func (ch *Channel) sendBytes(bytes []byte) bool {
    select {
    case ch.sendQueue <- bytes:
        atomic.AddInt32(&ch.sendQueueSize, 1)
        return true
    case <-time.After(defaultSendTimeout):
        return false
    }
}
```

它们两个的区别是，前者尝试把待发送数据`bytes`放入`ch.sendQueue`时，如果能放进去，则返回`true`，否则马上失败，返回`false`，所以它是非阻塞的。而后者，如果放不进去（`sendQueue`已满，那边还没处理完），则等待`defaultSendTimeout`（值为`10`秒），然后才会失败。另外，`sendQueue`的容量默认为`1`。

到这里，我们其实已经知道比原是如何向其它节点请求区块数据，以及何时把信息发送出去。

本想在本篇中就把真正发送数据的代码也一起讲了，但是发现它的逻辑也相当复杂，所以就另开一篇讲吧。

再回到本文问题，再强调一下，我们前面说了，对于向peer请求区块数据，有两种情况：一种是简单的不考虑分叉的，另一种是复杂的考虑分叉的。在本文只考虑了简单的情况，在这种情况下，所谓的`bestHeight`就是指的最高的那个区块的高度，而在复杂情况下，它就不一定了。这就留待以后我们再详细讨论，本文的问题就算是回答完毕了。

---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！
