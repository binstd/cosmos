---
id: bytomanalysis-howto-reply
title: 收到“请求区块数据”的信息后如何应答
permalink: docs/bytomanalysis-howto-reply.html
layout: docs
---

在上一篇，我们知道了比原是如何把“请求区块数据”的信息`BlockRequestMessage`发送给peer节点的，那么本文研究的重点就是，当peer节点收到了这个信息，它将如何应答？

那么这个问题如果细分的话，也可以分为三个小问题：

1. 比原节点是如何收到对方发过来的信息的？
2. 收到`BlockRequestMessage`后，将会给对方发送什么样的信息？
3. 这个信息是如何发送出去的？

我们先从第一个小问题开始。

比原节点是如何接收对方发过来的信息的？
-------------------------------

如果我们在代码中搜索`BlockRequestMessage`，会发现只有在`ProtocolReactor.Receive`方法中针对该信息进行了应答。那么问题的关键就是，比原是如何接收对方发过来的信息，并且把它转交给`ProtocolReactor.Receive`的。

如果我们对前一篇《比原是如何把请求区块数据的信息发出去的》有印象的话，会记得比原在发送信息时，最后会把信息写入到`MConnection.bufWriter`中；与之相应的，`MConnection`还有一个`bufReader`，用于读取数据，它也是与`net.Conn`绑定在一起的：

[p2p/connection.go#L114-L118](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L114-L118)

```go
func NewMConnectionWithConfig(conn net.Conn, chDescs []*ChannelDescriptor, onReceive receiveCbFunc, onError errorCbFunc, config *MConnConfig) *MConnection {
    mconn := &MConnection{
        conn:        conn,
        bufReader:   bufio.NewReaderSize(conn, minReadBufferSize),
        bufWriter:   bufio.NewWriterSize(conn, minWriteBufferSize),
```

（其中`minReadBufferSize`的值为常量`1024`）

所以，要读取对方发来的信息，一定会读取`bufReader`。经过简单的搜索，我们发现，它也是在`MConnection.Start`中启动的：

[p2p/connection.go#L152-L159](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L152-L159)

```go
func (c *MConnection) OnStart() error {
    // ...
    go c.sendRoutine()
    go c.recvRoutine()
    // ...
}
```

其中的`c.recvRoutine()`就是我们本次所关注的。它上面的`c.sendRoutine`是用来发送的，是前一篇文章中我们关注的重点。

继续`c.recvRoutine()`：

[p2p/connection.go#L403-L502](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L403-L502)

```go
func (c *MConnection) recvRoutine() {
    // ...
    for {
        c.recvMonitor.Limit(maxMsgPacketTotalSize, atomic.LoadInt64(&c.config.RecvRate), true)

        // ...

        pktType := wire.ReadByte(c.bufReader, &n, &err)
        c.recvMonitor.Update(int(n))
        // ...

        switch pktType {
        // ...
        case packetTypeMsg:
            pkt, n, err := msgPacket{}, int(0), error(nil)
            wire.ReadBinaryPtr(&pkt, c.bufReader, maxMsgPacketTotalSize, &n, &err)
            c.recvMonitor.Update(int(n))
            // ...
            channel, ok := c.channelsIdx[pkt.ChannelID]
            // ...
            msgBytes, err := channel.recvMsgPacket(pkt)
            // ...
            if msgBytes != nil {
                // ...
                c.onReceive(pkt.ChannelID, msgBytes)
            }
            // ...
        }
    }
    // ...
}
```

经过简化以后，这个方法分成了三块内容：

1. 第一块就限制接收速率，以防止恶意结点突然发送大量数据把节点撑死。跟发送一样，它的限制是`500K/s`
2. 第二块是从`c.bufReader`中读取出下一个数据包的类型。它的值目前有三个，两个跟心跳有关：`packetTypePing`和`packetTypePong`，另一个表示是正常的信息数据类型`packetTypeMsg`，也是我们需要关注的
3. 第三块就是继续从`c.bufReader`中读取出完整的数据包，然后根据它的`ChannelID`找到相应的channel去处理它。`ChannelID`有两个值，分别是`BlockchainChannel`和`PexChannel`，我们目前只需要关注前者即可，它对应的reactor是`ProtocolReactor`。当最后调用`c.onReceive(pkt.ChannelID, msgBytes)`时，读取的二进制数据`msgBytes`就会被`ProtocolReactor.Receive`处理

我们的重点是看第三块内容。首先是`channel.recvMsgPacket(pkt)`，即通道是怎么从packet包里读取到相应的二进制数据的呢？

[p2p/connection.go#L667-L682](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/connection.go#L667-L682)

```go
func (ch *Channel) recvMsgPacket(packet msgPacket) ([]byte, error) {
    // ...
    ch.recving = append(ch.recving, packet.Bytes...)
    if packet.EOF == byte(0x01) {
        msgBytes := ch.recving
        // ...
        ch.recving = ch.recving[:0]
        return msgBytes, nil
    }
    return nil, nil
}
```

这个方法我去掉了一些错误检查和关于性能方面的注释，有兴趣的同学可以点接上方的源代码查看，这里就忽略了。

这段代码主要是利用了一个叫`recving`的通道，把`packet`中持有的字节数组加到它后面，然后再判断该packet是否代表整个信息结束了，如果是的话，则把`ch.recving`的内容完整返回，供调用者处理；否则的话，返回一个`nil`，表示还没拿完，暂时处理不了。在前一篇文章中关于发送数据的地方可以与这里对应，只不过发送方要麻烦的多，需要三个通道`sendQueue`、`sending`和`send`才能实现，这边接收方就简单了。

然后回到前面的方法`MConnection.recvRoutine`，我们继续看最后的`c.onReceive`调用。这个`onReceive`实际上是一个由别人赋值给该channel的一个函数，它位于`MConnection`创建的地方：

[p2p/peer.go#L292-L310](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/peer.go#L292-L310)

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

逻辑也比较简单，就是当前面的`c.onReceive(pkt.ChannelID, msgBytes)`调用时，它会根据传入的`chID`找到相应的`Reactor`，然后执行其`Receive`方法。对于本文来说，就会进入到`ProtocolReactor.Receive`。

那我们继续看`ProtocolReactor.Receive`:

[netsync/protocol_reactor.go#L179-L247](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/protocol_reactor.go#L179-L247)

```go
func (pr *ProtocolReactor) Receive(chID byte, src *p2p.Peer, msgBytes []byte) {
    _, msg, err := DecodeMessage(msgBytes)
    // ...
    switch msg := msg.(type) {
    case *BlockRequestMessage:
        // ...
}
```

其中的`DecodeMessage(...)`就是把传入的二进制数据反序列化成一个`BlockchainMessage`对象，该对象是一个没有任何内容的`interface`，它有多种实现类型。我们在后面继续对该对象进行判断，如果它是`BlockRequestMessage`类型的信息，我们就会继续做相应的处理。处理的代码我在这里暂时省略了，因为它是属于下一个小问题的，我们先不考虑。

好像不知不觉我们就把第一个小问题的后半部分差不多搞清楚了。那么前半部分是什么？我们在前面说，读取`bufReader`的代码的起点是在`MConnection.Start`中，那么前半部分就是：比原从启动开始中，是在什么情况下怎样一步步走到`MConnection.Start`的呢？

好在前半部分的问题我们在前一篇文章《比原是如何把请求区块数据的信息发出去的》中进行了专门的讨论，这里就不讲了，有需要的话可以再过去看一下（可以先看最后“总结”那一小节）。

下面我们进入第二个小问题：

收到`BlockRequestMessage`后，将会给对方发送什么样的信息？
--------------------------------------------------

这里就是接着前面的`ProtocolReactor.Receive`继续向下讲了。首先我们再贴一下它的较完整的代码：

[netsync/protocol_reactor.go#L179-L247](https://github.com/freewind/bytom-v1.0.1/blob/master/netsync/protocol_reactor.go#L179-L247)

```go
func (pr *ProtocolReactor) Receive(chID byte, src *p2p.Peer, msgBytes []byte) {
    _, msg, err := DecodeMessage(msgBytes)
    // ...

    switch msg := msg.(type) {
    case *BlockRequestMessage:
        var block *types.Block
        var err error
        if msg.Height != 0 {
            block, err = pr.chain.GetBlockByHeight(msg.Height)
        } else {
            block, err = pr.chain.GetBlockByHash(msg.GetHash())
        }
        // ...
        response, err := NewBlockResponseMessage(block)
        // ...
        src.TrySend(BlockchainChannel, struct{ BlockchainMessage }{response})
    // ...
}
```

可以看到，逻辑还是比较简单的，即根据对方发过来的`BlockRequestMessage`中指定的`height`或者`hash`信息，在本地的区块链数据中找到相应的block，组成`BlockResponseMessage`发过去就行了。

其中`chain.GetBlockByHeight(...)`和`chain.GetBlockByHash(...)`如果详细说明的话，需要深刻理解区块链数据在比原节点中是如何保存的，我们在本文先不讲，等到后面专门研究。

在这里，我觉得我们只需要知道我们会查询区块数据并且构造出一个`BlockResponseMessage`，再通过`BlockchainChannel`这个通道发送出去就可以了。

最后一句代码中调用了`src.TrySend`方法，它是把信息向对方peer发送过去。（其中的`src`就是指的对方peer）

那么，它到底是怎么发送出去的呢？下面我们进入最后一个小问题：

这个`BlockResponseMessage`信息是如何发送出去的？
----------------------------------------------

我们先看看`peer.TrySend`代码：

[p2p/peer.go#L242-L247](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/peer.go#L242-L247)

```go
func (p *Peer) TrySend(chID byte, msg interface{}) bool {
    if !p.IsRunning() {
        return false
    }
    return p.mconn.TrySend(chID, msg)
}
```

它在内部将会调用`MConnection.TrySend`方法，其中`chID`是`BlockchainChannel`，也就是它对应的Reactor是`ProtocolReactor`。

再接着就是我们熟悉的`MConnection.TrySend`，由于它在前一篇文章中进行了全面的讲解，在本文就不提了，如果需要可以过去翻看一下。

那么今天的问题就算是解决啦。

到这里，我们总算能够完整的理解清楚，当我们向一个比原节点请求“区块数据”，我们这边需要怎么做，对方节点又需要怎么做了。


---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！
