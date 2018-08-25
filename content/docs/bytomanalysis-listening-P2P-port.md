---
id: bytomanalysis-listening-P2P-port
title: 监听p2p端口
permalink: docs/bytomanalysis-listening-P2P-port.html
layout: docs
---

我们知道，在使用`bytomd init --chain_id mainnet/testnet/solonet`初始化比原的时候，它会根据给定的`chain_id`的不同，使用不同的端口(参看[config/toml.go#L29](https://github.com/freewind/bytom-v1.0.1/blob/master/config/toml.go#L29-L45))：

1. `mainnet`（连接到主网）: `46657`
2. `testnet`（连接到测试网）: `46656`
3. `solonet`（本地单独节点）: `46658`

对于我来说，由于只需要对本地运行的一个比原节点进行分析，所以可以采用第3个`chain_id`，即`solonet`。这样它启动之后，不会与其它的节点主动连接，可以减少其它节点对于我们的干扰。

所以在启动的时候，我的命令是这样的：

```
cd cmd/bytomd
./bytomd init --chain_id solonet
./bytomd node
```

它就会监听`46658`端口，等待其它节点的连接。

### 连上看看

如果这时我们使用`telnet`来连接其`46658`端口，成功连接上之后，可以看到它会发给我们一些乱码，大概如下：

```
$ telnet localhost 46658
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
ט�S��%�z?��_�端��݂���U[e
```

我们也许会好奇，它发给我们的到底是什么？

但是这个问题留待下次回答，因为首先，比原节点必须能够监听这个端口，我们才能连上。所以这次我们的问题是：

比原在代码中是如何监听这个端口的?
---------------------------

### 端口已经写在`config.toml`中

在前面，当我们使用`./bytomd init --chain_id solonet`初始化比原以后，比原会在本地的数据目录中生成一个`config.toml`的配置文件，内容大约如下：

```ini
# This is a TOML config file.
# For more information, see https://github.com/toml-lang/toml
fast_sync = true
db_backend = "leveldb"
api_addr = "0.0.0.0:9888"
chain_id = "solonet"
[p2p]
laddr = "tcp://0.0.0.0:46658"
seeds = ""
```

其中`[p2p]`下面的`laddr`，就是该节点监听的地址和端口。

对于`laddr = "tcp://0.0.0.0:46658"`，它是意思是：

1. 使用的是`tcp`协议
2. 监听的ip是`0.0.0.0`，是指监听本机所有ip地址。这样该节点既允许本地访问，也允许外部主机访问。如果你只想让它监听某一个ip，手动修改该配置文件即可
3. `46658`，就是我们在这个问题中关注的端口了，它与该节点与其它节点交互数据使用的端口

比原在监听这个端口的时候，并不是如我最开始预期的直接调用`net.Listen`监听它。实际的过程要比这个复杂，因为比原设计了一个叫`Switch`的对象，用来统一管理与外界相关的事件，包括监听、连接、发送消息等。而`Switch`这个对象，又是在`SyncManager`中创建的。

### 启动直到进入`Switch`

所以我们首先需要知道，比原在源代码中是如何启动，并且一步步走进了`Switch`的世界。

首先还是当我们`bytomd node`启动比原时，对应的入口函数如下：

[cmd/bytomd/main.go#L54](https://github.com/freewind/bytom/blob/master/cmd/bytomd/main.go#L54)

```go
func main() {
    cmd := cli.PrepareBaseCmd(commands.RootCmd, "TM", os.ExpandEnv(config.DefaultDataDir()))
    cmd.Execute()
}
```

它又会根据传入的`node`参数，运行下面的函数：

[cmd/bytomd/commands/run_node.go#L41](https://github.com/freewind/bytom/blob/master/cmd/bytomd/commands/run_node.go#L41)

```go
func runNode(cmd *cobra.Command, args []string) error {
    // Create & start node
    n := node.NewNode(config)
    // ...
}
```

我们需要关注的是`node.NewNode(config)`函数，因为是在它里面创建了`SyncManager`：

[node/node.go#L59](https://github.com/freewind/bytom/blob/master/node/node.go#L59)

```go
func NewNode(config *cfg.Config) *Node {
    // ...
    syncManager, _ := netsync.NewSyncManager(config, chain, txPool, newBlockCh)
    // ...
}
```

在创建`SyncManager`的时候，又创建了`Switch`:

[netsync/handle.go#L42](https://github.com/freewind/bytom/blob/master/netsync/handle.go#L42)

```go
func NewSyncManager(config *cfg.Config, chain *core.Chain, txPool *core.TxPool, newBlockCh chan *bc.Hash) (*SyncManager, error) {
    // ...
    manager.sw = p2p.NewSwitch(config.P2P, trustHistoryDB)

    // ...
    protocolReactor := NewProtocolReactor(chain, txPool, manager.sw, manager.blockKeeper, manager.fetcher, manager.peers, manager.newPeerCh, manager.txSyncCh, manager.dropPeerCh)
    manager.sw.AddReactor("PROTOCOL", protocolReactor)

    // Create & add listener
    p, address := protocolAndAddress(manager.config.P2P.ListenAddress)
    l := p2p.NewDefaultListener(p, address, manager.config.P2P.SkipUPNP, nil)
    manager.sw.AddListener(l)

    // ...
}
```

这里需要注意一下，上面创建的`protocolReactor`对象，是用来处理当有节点连接上端口后，双方如何交互的事情。跟这次问题“监听端口”没有直接关系，但是这里也可以注意一下。

然后又创建了一个`DefaultListener`对象，而监听端口的动作，就是在它里面发生的。Listener创建之后，将会添加到`manager.sw`（即`Switch`）中，用于在那边进行外界数据与事件的交互。

### 监听端口

`NewDefaultListener`中做的事情比较多，所以我们把它分成几块说：

[p2p/listener.go#L52](https://github.com/freewind/bytom/blob/master/p2p/listener.go#L52)

```go
func NewDefaultListener(protocol string, lAddr string, skipUPNP bool, logger tlog.Logger) Listener {
    // Local listen IP & port
    lAddrIP, lAddrPort := splitHostPort(lAddr)

    // Create listener
    var listener net.Listener
    var err error
    for i := 0; i < tryListenSeconds; i++ {
        listener, err = net.Listen(protocol, lAddr)
        if err == nil {
            break
        } else if i < tryListenSeconds-1 {
            time.Sleep(time.Second * 1)
        }
    }
    if err != nil {
        cmn.PanicCrisis(err)
    }

    // ...
```

上面这部分就是真正监听的代码了。通过Go语言提供的`net.Listen`函数，监听了指定的地址。另外，在监听的时候，进行了多次尝试，因为当一个刚刚被使用的端口被放开后，还需要一小段时间才能真正释放，所以这里需要多尝试几次。

其中`tryListenSeconds`是一个常量，值为`5`，也就是说，大约会尝试5秒钟，要是都绑定不上，才会真正失败，抛出错误。

后面省略了一些代码，主要是用来获取当前监听的实际ip以及外网ip，并记录在日志中。本想在这里简单讲讲，但是发现还有点麻烦，所以打算放在后面专开一个问题。

其实本次问题到这里就已经结束了，因为已经完成了“监听”。但是后面还有一些初始化操作，是为了让比原可以跟连接上该端口的节点进行交互，也值得在这里讲讲。

接着刚才的方法，最后的部分是：

```go
    dl := &DefaultListener{
        listener:    listener,
        intAddr:     intAddr,
        extAddr:     extAddr,
        connections: make(chan net.Conn, numBufferedConnections),
    }
    dl.BaseService = *cmn.NewBaseService(logger, "DefaultListener", dl)
    dl.Start() // Started upon construction
    return dl
}
```

需要注意的是`connections`，它是一个带有缓冲的channel（`numBufferedConnections`值为`10`），用来存放连接上该端口的连接对象。这些操作将在后面的`dl.Start()`中执行。

`dl.Start()`将调用`DefaultListener`对应的`OnStart`方法，如下：

[p2p/listener.go#L114](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/listener.go#L114)

```go
func (l *DefaultListener) OnStart() error {
    l.BaseService.OnStart()
    go l.listenRoutine()
    return nil
}
```

其中的`l.listenRoutine`，就是执行前面所说的向`connections` channel里放入连接的函数:

[p2p/listener.go#L126](https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/listener.go#L126)

```go
func (l *DefaultListener) listenRoutine() {
    for {
        conn, err := l.listener.Accept()
        // ...
        l.connections <- conn
    }

    // Cleanup
    close(l.connections)

    // ...
}
```

而`Switch`在`SyncManager`启动的时候会被启动，在它的`OnStart`方法中，会拿到所有Listener（即监听端口的对象）中`connections`channel中的连接，与它们交互。

https://github.com/freewind/bytom-v1.0.1/blob/master/p2p/switch.go#L498
```go
func (sw *Switch) listenerRoutine(l Listener) {
    for {
        inConn, ok := <-l.Connections()
        if !ok {
            break
        }
        // ...
        
        err := sw.addPeerWithConnectionAndConfig(inConn, sw.peerConfig)
        // ...
    }
```

其中`sw.addPeerWithConnectionAndConfig`就是与对应节点进行交互的逻辑所在，但是这已经超出了本次问题的范畴，下次再讲。

到此为止，本次的问题，应该已经讲清楚了。

---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！
