---
id: bytomanalysis-howto-senddata
title: 把请求区块数据的信息发出去
permalink: docs/bytomanalysis-howto-senddata.html
layout: docs
---

在前一篇中，我们说到，当比原向其它节点请求区块数据时，`BlockKeeper`会发送一个`BlockRequestMessage`把需要的区块`height`告诉对方，并把该信息对应的二进制数据放入`ProtocolReactor`对应的`sendQueue`通道中，等待发送。而具体的发送细节，由于逻辑比较复杂，所以在前一篇中并未详解，放到本篇中。

由于`sendQueue`是一个通道，数据放进去后，到底是由谁在什么情况下取走并发送，`BlockKeeper`这边是不知道的。经过我们在代码中搜索，发现只有一个类型会直接监视`sendQueue`中的数据，它就是前文出现的`MConnection`。`MConnection`的对象在它的`OnStart`方法中，会监视`sendQueue`中的数据，然后，等发现数据时，会将之取走并放入一个叫`sending`的通道里。

事情变得有点复杂了：

1. 由前篇我们知道，一个`MConnection`对应了一个与peer的连接，而比原节点之间建立连接的情况又有多种：比如主动连接别的节点，或者别的节点主动连上我
2. 放入通道`sending`之后，我们还需要知道又是谁在什么情况下会监视`sending`，取走它里面的数据
3. `sending`中的数据被取走后，又是如何被发送到其它节点的呢？

还是像以前一样，遇到复杂的问题，我们先通过“相互独立，完全穷尽”的原则，把它分解成一个个小问题，然后依次解决。

那么首先我们需要弄清楚的是：

比原在什么情况下，会创建`MConnection`的对象并调用其`OnStart`方法？
----------------------------------------------------------

（从而我们知道`sendQueue`中的数据是如何被监视的）

经过分析，我们发现`MConnection`的启动，只出现在一个地方，即`Peer`的`OnStart`方法中。那么就这个问题就变成了：比原在什么情况下，会创建`Peer`的对象并调用其`OnStart`方法？

再经过一番折腾，终于确定，在比原中，在下列4种情况`Peer.OnStart`方法最终会被调用：

1. 比原节点启动后，主动去连接配置文件指定的种子节点、以及本地数据目录中`addrbook.json`中保存的节点的时候
2. 比原监听本地p2p端口后，有别的节点连上来的时候
3. 启动`PEXReactor`，并使用它自己的协议与当前连接上的节点进行通信的时候
4. 在一个没有用上的`Switch.Connect2Switches`方法中（可忽略）

第4种情况我们完全忽略。第3种情况中，由于`PEXReactor`会使用类似于BitTorrent的文件分享协议与其它节点分享数据，逻辑比较独立，算是一种辅助作用，我们也暂不考虑。这样我们就只需要分析前两种情况了。

### 比原节点启动时，是如何主动连接其它节点，并最终调用了`MConnection.OnStart`方法的？

首先我们快速走到`SyncManager.Start`方法:

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
}
```

然后我们将进入`netStart()`方法。在这个方法中，比原将主动连接其它节点:

```go
func (sm *SyncManager) netStart() error {
    // ...
    if sm.config.P2P.Seeds != "" {
        // dial out
        seeds := strings.Split(sm.config.P2P.Seeds, ",")
        if err := sm.DialSeeds(seeds); err != nil {
            return err
        }
    }

    return nil
}
```

这里出现的`sm.config.P2P.Seeds`，对应的就是本地数据目录中`config.toml`中的`p2p.seeds`中的种子结点。

接着通过`sm.DialSeeds`去主动连接每个种子：

[netsync/handle.go#L229-L231](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/handle.go#L229-L231)
```go
func (sm *SyncManager) DialSeeds(seeds []string) error {
    return sm.sw.DialSeeds(sm.addrBook, seeds)
}
```

[p2p/switch.go#L311-L340](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L311-L340)

```go
func (sw *Switch) DialSeeds(addrBook *AddrBook, seeds []string) error {
    // ...
    for i := 0; i < len(perm)/2; i++ {
        j := perm[i]
        sw.dialSeed(netAddrs[j])
    }
   // ...
}
```

[p2p/switch.go#L342-L349](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L342-L349)

```go
func (sw *Switch) dialSeed(addr *NetAddress) {
    peer, err := sw.DialPeerWithAddress(addr, false)
    // ...
}
```

[p2p/switch.go#L351-L392](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L351-L392)
```go
func (sw *Switch) DialPeerWithAddress(addr *NetAddress, persistent bool) (*Peer, error) {
    // ...
    peer, err := newOutboundPeerWithConfig(addr, sw.reactorsByCh, sw.chDescs, sw.StopPeerForError, sw.nodePrivKey, sw.peerConfig)
    // ...
    err = sw.AddPeer(peer)
    // ...
}
```

先是通过`newOutboundPeerWithConfig`创建了`peer`，然后把它加入到`sw`（即`Switch`对象）中。

[p2p/switch.go#L226-L275](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L226-L275)

```go
func (sw *Switch) AddPeer(peer *Peer) error {
    // ...
    // Start peer
    if sw.IsRunning() {
        if err := sw.startInitPeer(peer); err != nil {
            return err
        }
    }
    // ...
}
```

在`sw.startInitPeer`中，将会调用`peer.Start`：

[p2p/switch.go#L300-L308](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L300-L308)

```go
func (sw *Switch) startInitPeer(peer *Peer) error {
    peer.Start()
    // ...
}
```

而`peer.Start`对应了`Peer.OnStart`，最后就是:

[p2p/peer.go#L207-L211](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/peer.go#L207-L211)
```go
func (p *Peer) OnStart() error {
    p.BaseService.OnStart()
    _, err := p.mconn.Start()
    return err
}
```

可以看到，在这里调用了`mconn.Start`，终于找到了。总结一下就是：

- `Node.Start` -> `SyncManager.Start` -> `SyncManager.netStart` ->  `Switch.DialSeeds` -> `Switch.AddPeer` -> `Switch.startInitPeer` -> `Peer.OnStart` -> `MConnection.OnStart`

那么，第一种主动连接别的节点的情况就到这里分析完了。下面是第二种情况：

### 当别的节点连接到本节点时，比原是如何走到`MConnection.OnStart`方法这一步的？

比原节点启动后，会监听本地的p2p端口，等待别的节点连接上来。那么这个流程又是什么样的呢？

由于比原节点的启动流程在目前的文章中已经多次出现，这里就不贴了，我们直接从`Switch.OnStart`开始（它是在`SyncManager`启动的时候启动的）：

[p2p/switch.go#L186-L185](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L186-L185)

```go
func (sw *Switch) OnStart() error {
    // ...
    for _, peer := range sw.peers.List() {
        sw.startInitPeer(peer)
    }
    
    // Start listeners
    for _, listener := range sw.listeners {
        go sw.listenerRoutine(listener)
    }
    // ...
}
```

这个方法经过省略以后，还剩两块代码，一块是`startInitPeer(...)`，一块是`sw.listenerRoutine(listener)`。

如果你刚才在读前一节时留意了，就会发现，`startInitPeer(...)`方法马上就会调用`Peer.Start`。然而在这里需要说明的是，经过我的分析，发现这块代码实际上没有起到任何作用，因为在当前这个时刻，`sw.peers`总是空的，它里面还没有来得及被其它的代码添加进peer。所以我觉得它可以删掉，以免误导读者。（提了一个issue，参见[#902](https://github.com/Bytom/bytom/issues/902)）

第二块代码，`listenerRoutine`，如果你还有印象的话，它就是用来监听本地p2p端口的，在前面“比原是如何监听p2p端口的”一文中有详细的讲解。

我们今天还是需要再挖掘一下它，看看它到底是怎么走到`MConnection.OnStart`的：

[p2p/switch.go#L498-L536](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L498-L536)

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

这里的`l`就是监听本地p2p端口的Listener。通过一个`for`循环，拿到连接到该端口的节点的连接，生成新peer。

```go
func (sw *Switch) addPeerWithConnectionAndConfig(conn net.Conn, config *PeerConfig) error {
    // ...
    peer, err := newInboundPeerWithConfig(conn, sw.reactorsByCh, sw.chDescs, sw.StopPeerForError, sw.nodePrivKey, config)
    // ...
    if err = sw.AddPeer(peer); err != nil {
        // ...
    }
    // ...
}
```

生成新的peer之后，调用了`Switch`的`AddPeer`方法。到了这里，就跟前一节一样了，在`AddPeer`中将调用`sw.startInitPeer(peer)`，然后调用`peer.Start()`，最后调用了`MConnection.OnStart()`。由于代码一模一样，就不贴出来了。

总结一下，就是：

- `Node.Start` -> `SyncManager.Start` -> `SyncManager.netStart` -> `Switch.OnStart` -> `Switch.listenerRoutine` -> `Switch.addPeerWithConnectionAndConfig` -> `Switch.AddPeer` -> `Switch.startInitPeer` -> `Peer.OnStart` -> `MConnection.OnStart`

那么，第二种情况我们也分析完了。

不过到目前为止，我们只解决了这次问题中的第一个小问题，即：我们终于知道了比原代码会在什么情况来启动一个`MConnection`，从而监视`sendQueue`通道，把要发送的信息数据，转到了`sending`通道中。

那么，我们进入下一个小问题：

数据放入通道`sending`之后，谁又会来取走它们呢？
----------------------------------------

经过分析之后，发现通道`sendQueue`和`sending`都属于类型`Channel`，只不过两者作用不同。`sendQueue`是用来存放待发送的完整的信息数据，而`sending`更底层一些，它持有的数据可能会被分成多个块发送。如果只有`sendQueue`一个通道，那么很难实现分块的操作的。

而`Channel`的发送是由`MConnection`来调用的，幸运的是，当我们一直往回追溯下去，发现竟走到了`MConnection.OnStart`这里。也就是说，我们在这个小问题中，研究的正好是前面两个链条后面的部分：

- `Node.Start` -> `SyncManager.Start` -> `SyncManager.netStart` ->  `Switch.DialSeeds` -> `Switch.AddPeer` -> `Switch.startInitPeer` -> `Peer.OnStart` -> `MConnection.OnStart` -> `???`

- `Node.Start` -> `SyncManager.Start` -> `SyncManager.netStart` -> `Switch.OnStart` -> `Switch.listenerRoutine` -> `Switch.addPeerWithConnectionAndConfig` -> `Switch.AddPeer` -> `Switch.startInitPeer` -> `Peer.OnStart` -> `MConnection.OnStart` -> `???`

也就是上面的`???`部分。

那么我们就直接从`MConnection.OnStart`开始：

[p2p/connection.go#L152-L159](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L152-L159)

```go
func (c *MConnection) OnStart() error {
    // ...
    go c.sendRoutine()
    // ...
}
```

`c.sendRoutine()`方法就是我们需要的。当`MConnection`启动以后，就会开始进行发送操作（等待数据到来)。它的代码如下：

[p2p/connection.go#L289-L343](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L289-L343)

```go
func (c *MConnection) sendRoutine() {
    // ...
        case <-c.send:
            // Send some msgPackets
            eof := c.sendSomeMsgPackets()
            if !eof {
                // Keep sendRoutine awake.
                select {
                case c.send <- struct{}{}:
                default:
                }
            }
        }
    // ...
}
```

这个方法本来很长，只是我们省略掉了很多无关的代码。里面的`c.sendSomeMsgPackets()`就是我们要找的，但是，我们突然发现，怎么又出来了一个`c.send`通道？它又有什么用？而且看起来好像只有当这个通道里有东西的时候，我们才会去调用`c.sendSomeMsgPackets()`，似乎像是一个铃铛一样用来提醒我们。

那么`c.send`什么时候会有东西呢？检查了代码之后，发现在以下3个地方：

[p2p/connection.go#L206-L239](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L206-L239)

```go
func (c *MConnection) Send(chID byte, msg interface{}) bool {
    // ...
    success := channel.sendBytes(wire.BinaryBytes(msg))
    if success {
        // Wake up sendRoutine if necessary
        select {
        case c.send <- struct{}{}:
        // ..
}
```

[p2p/connection.go#L243-L271](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L243-L271)

```go
func (c *MConnection) TrySend(chID byte, msg interface{}) bool {
    // ...
    ok = channel.trySendBytes(wire.BinaryBytes(msg))
    if ok {
        // Wake up sendRoutine if necessary
        select {
        case c.send <- struct{}{}:
        // ...
}
```

[p2p/connection.go#L289-L343](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L289-L343)

```go
func (c *MConnection) sendRoutine() {
    // ....
        case <-c.send:
            // Send some msgPackets
            eof := c.sendSomeMsgPackets()
            if !eof {
                // Keep sendRoutine awake.
                select {
                case c.send <- struct{}{}:
                // ...
}
```

如果我们对前一篇文章还有印象，就会记得`channel.trySendBytes`是在我们想给对方节点发信息时调用的，调用完以后，它会把信息对应的二进制数据放入到`channel.sendQueue`通道（所以才有了本文）。`channel.sendBytes`我们目前虽然还没用到，但是它也应该是类似的。在它们两个调用完之后，它们都会向`c.send`通道里放入一个数据，用来通知`Channel`有数据可以发送了。

而第三个`sendRoutine()`就是我们刚刚走到的地方。当我们调用`c.sendSomeMsgPackets()`发送了`sending`中的一部分之后，如果还有剩余的，则继续向`c.send`放个数据，提醒可以继续发送。

那到目前为止，发送数据涉及到的Channel就有三个了，分别是`sendQueue`、`sending`和`send`。之所以这么复杂，根本原因就是想把数据分块发送。

为什么要分块发送呢？这是因为比原希望能控制发送速率，让节点之间的网速能保持在一个合理的水平。如果不限制的话，一下子发出大量的数据，一是可能会让接收者来不及处理，二是有可能会被恶意节点利用，请求大量区块数据把带宽占满。

担心`sendQueue`、`sending`和`send`这三个通道不太好理解，我想到了一个“烧鸭店”的比喻，来理解它们：

- `sendQueue`就像是用来挂烤好的烧鸭的勾子，可以有多个（但对于比原来说，默认只有一个，因为`sendQueue`的容量默认为`1`），当有烧鸭烤好以后，就挂在勾子上；
- `sending`是砧板，可以把烧鸭从`sendQueue`勾子上取下来一只，放在上面切成块，等待装盘，一只烧鸭可能可以装成好几盘；
- 而`send`是铃铛，当有人点单后，服务员就会按一下铃铛，厨师就从`sending`砧板上拿几块烧鸭放在小盘中放在出餐口。由于厨师非常忙，每次切出一盘后都可能会去做别的事情，而忘了`sending`砧板上还有烧鸭没装盘，所以为了防止自己忘记，他每切出一盘之后，都会看一眼`sending`砧板，如果还有肉，就会按一下铃铛提醒自己继续装盘。

好了，理解了`send`后，我们就可以回到主线，继续看`c.sendSomeMsgPackets()`的代码了：

[p2p/connection.go#L347-L360](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L347-L360)

```go
func (c *MConnection) sendSomeMsgPackets() bool {
    // Block until .sendMonitor says we can write.
    // Once we're ready we send more than we asked for,
    // but amortized it should even out.
    c.sendMonitor.Limit(maxMsgPacketTotalSize, atomic.LoadInt64(&c.config.SendRate), true)

    // Now send some msgPackets.
    for i := 0; i < numBatchMsgPackets; i++ {
        if c.sendMsgPacket() {
            return true
        }
    }
    return false
}
```

`c.sendMonitor.Limit`的作用是限制发送速率，其中`maxMsgPacketTotalSize`即每个packet的最大长度为常量`10240`，第二个参数是预先指定的发送速率，默认值为`500KB/s`，第三个参数是说，当实际速度过大时，是否暂停发送，直到变得正常。

经过限速的调整后，后面一段就可以正常发送数据了，其中的`c.sendMsgPacket`是我们继续要看的方法：

[p2p/connection.go#L363-L398](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L363-L398)

```go
func (c *MConnection) sendMsgPacket() bool {
    // ...
    n, err := leastChannel.writeMsgPacketTo(c.bufWriter)
    // ..
    c.sendMonitor.Update(int(n))
    // ...
    return false
}
```

这个方法最前面我省略了一大段代码，其作用是检查多个channel，结合它们的优先级和已经发的数据量，找到当前最需要发送数据的那个channel，记为`leastChannel`。

然后就是调用`leastChannel.writeMsgPacketTo(c.bufWriter)`，把当前要发送的一块数据，写到`bufWriter`中。这个`bufWriter`就是真正与连接对象绑定的一个缓存区，写入到它里面的数据，会被Go发送出去。它的定义是在创建`MConnection`的地方：

[p2p/connection.go#L114-L118](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L114-L118)

```go
func NewMConnectionWithConfig(conn net.Conn, chDescs []*ChannelDescriptor, onReceive receiveCbFunc, onError errorCbFunc, config *MConnConfig) *MConnection {
    mconn := &MConnection{
        conn:        conn,
        bufReader:   bufio.NewReaderSize(conn, minReadBufferSize),
        bufWriter:   bufio.NewWriterSize(conn, minWriteBufferSize),
```

其中`minReadBufferSize`为`1024`，`minWriteBufferSize`为`65536`。

数据写到`bufWriter`以后，我们就不需要关心了，交给Go来操作了。

在`leastChannel.writeMsgPacketTo(c.bufWriter)`调用完以后，后面会更新`c.sendMonitor`，这样它才能继续正确的限速。

这时我们已经知道数据是怎么发出去的了，但是我们还没有找到是谁在监视`sending`里的数据，那让我们继续看`leastChannel.writeMsgPacketTo`：

[p2p/connection.go#L655-L663](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L655-L663)

```go
func (ch *Channel) writeMsgPacketTo(w io.Writer) (n int, err error) {
    packet := ch.nextMsgPacket()
    wire.WriteByte(packetTypeMsg, w, &n, &err)
    wire.WriteBinary(packet, w, &n, &err)
    if err == nil {
        ch.recentlySent += int64(n)
    }
    return
}
```

其中的`ch.nextMsgPacket()`是取出下一个要发送的数据块，那么是从哪里取出呢？是从`sending`吗？

其后的代码是把数据块对象变成二进制，放入到前面的`bufWriter`中发送。

继续`ch.nextMsgPacket()`：

[p2p/connection.go#L638-L651](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L638-L651)

```go
func (ch *Channel) nextMsgPacket() msgPacket {
    packet := msgPacket{}
    packet.ChannelID = byte(ch.id)
    packet.Bytes = ch.sending[:cmn.MinInt(maxMsgPacketPayloadSize, len(ch.sending))]
    if len(ch.sending) <= maxMsgPacketPayloadSize {
        packet.EOF = byte(0x01)
        ch.sending = nil
        atomic.AddInt32(&ch.sendQueueSize, -1) // decrement sendQueueSize
    } else {
        packet.EOF = byte(0x00)
        ch.sending = ch.sending[cmn.MinInt(maxMsgPacketPayloadSize, len(ch.sending)):]
    }
    return packet
}
```

终于看到`sending`了。从这里可以看出，`sending`的确是放着很多块鸭肉的砧板，而`packet`就是一个小盘，所以需要从先`sending`中拿出不超过指定长度的数据放到`packet`中，然后判断`sending`里还有没有剩下的。如果有，则`packet`的`EOF`值为`0x00`，否则为`0x01`，这样调用者就知道数据有没有发完，还需不需要去按那个叫`send`的铃。

那么到这里为止，我们就知道原来还是Channel自己在关注`sending`，并且为了限制发送速度，需要把它切成一个个小块。

最后就我们的第三个小问题了，其实我们刚才在第二问里已经弄清楚了。

`sending`中的数据被取走后，又是如何被发送到其它节点的呢？
------------------------------------------------

答案就是，`sending`中的数据被分成一块块取出来后，会放入到`bufWriter`中，就直接被Go的`net.Conn`对象发送出去了。到这一层面，就不需要我们再继续深入了。

总结
---

由于本篇中涉及的方法调用比较多，可能看完都乱了，所以在最后，我们前面调用链补充完整，放在最后：

- `Node.Start` -> `SyncManager.Start` -> `SyncManager.netStart` ->  `Switch.DialSeeds` -> `Switch.AddPeer` -> `Switch.startInitPeer` -> `Peer.OnStart` -> `MConnection.OnStart` -> `...`

- `Node.Start` -> `SyncManager.Start` -> `SyncManager.netStart` -> `Switch.OnStart` -> `Switch.listenerRoutine` -> `Switch.addPeerWithConnectionAndConfig` -> `Switch.AddPeer` -> `Switch.startInitPeer` -> `Peer.OnStart` -> `MConnection.OnStart` -> `...`

然后是：

- `MConnection.sendRoutine` -> `MConnection.send` -> `MConnection.sendSomeMsgPackets` -> `MConnection.sendMsgPacket` -> `MConnection.writeMsgPacketTo` -> `MConnection.nextMsgPacket` -> `MConnection.sending`

到了最后，我的感觉就是，一个复杂问题最开始看起来很可怕，但是一旦把它分解成小问题之后，每次只关注一个，各个击破，好像就没那么复杂了。

---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！
