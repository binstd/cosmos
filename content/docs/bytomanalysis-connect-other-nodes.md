---
id: bytomanalysis-connect-other-nodes
title: 连接别的节点
permalink: docs/bytomanalysis-connect-other-nodes.html
layout: docs
---

最开始我对于这个问题一直有个疑惑：区块链是一个分布式的网络，那么一个节点启动后，它怎么知道去哪里找别的节点从而加入网络呢？

看到代码之后，我才明白，原来在代码中硬编码了一些种子地址，这样在启动的时候，可以先通过种子地址加入网络。虽然整个网络是分布式的，但是最开始还是需要一定的中心化。

### 预编码内容

对于配置文件`config.toml`，比原的代码中硬编码了配置文件内容：

[config/toml.go#L22-L45](https://github.com/freewind/bytom/blob/master/config/toml.go#L22-L45)

```go
var defaultConfigTmpl = `# This is a TOML config file.
# For more information, see https://github.com/toml-lang/toml
fast_sync = true
db_backend = "leveldb"
api_addr = "0.0.0.0:9888"
`

var mainNetConfigTmpl = `chain_id = "mainnet"
[p2p]
laddr = "tcp://0.0.0.0:46657"
seeds = "45.79.213.28:46657,198.74.61.131:46657,212.111.41.245:46657,
47.100.214.154:46657,47.100.109.199:46657,47.100.105.165:46657"
`

var testNetConfigTmpl = `chain_id = "testnet"
[p2p]
laddr = "tcp://0.0.0.0:46656"
seeds = "47.96.42.1:46656,172.104.224.219:46656,45.118.132.164:46656"
`

var soloNetConfigTmpl = `chain_id = "solonet"
[p2p]
laddr = "tcp://0.0.0.0:46658"
seeds = ""
`
```

可以看出，对于不同的`chain_id`，预设的种子是不同的。

当然，如果我们自己知道某些节点的地址，也可以在初始化生成`config.toml`后，手动修改该文件添加进去。

### 启动`syncManager`

那么，比原在代码中是使用这些种子地址并连接它们的呢？关键在于，连接的代码位于`SyncManager`中，所以我们要找到启动`syncManager`的地方。

首先，当我们使用`bytomd node`启动后，下面的函数将被调用：

[cmd/bytomd/commands/run_node.go#L41](https://github.com/freewind/bytom/blob/master/cmd/bytomd/commands/run_node.go#L41)

```go
func runNode(cmd *cobra.Command, args []string) error {
    // Create & start node
    n := node.NewNode(config)
    if _, err := n.Start(); err != nil {
        // ...
    }
    // ...
}
```

这里调用了`n.Start`，其中的`Start`方法，来自于`Node`所嵌入的`cmn.BaseService`：

[node/node.go#L39](https://github.com/freewind/bytom/blob/master/node/node.go#L39)

```go
type Node struct {
    cmn.BaseService
    // ...
}
```

所以`n.Start`对应的是下面这个方法：

[vendor/github.com/tendermint/tmlibs/common/service.go#L97](https://github.com/freewind/bytom/blob/master/vendor/github.com/tendermint/tmlibs/common/service.go#L97)

```go
func (bs *BaseService) Start() (bool, error) {
    // ...
    err := bs.impl.OnStart()
    // ...
}
```

在这里，由于`bs.impl`对应于`Node`，所以将继续调用`Node.OnStart()`:

[node/node.go#L169](https://github.com/freewind/bytom/blob/master/node/node.go#L169)

```go
func (n *Node) OnStart() error {
    // ...
    n.syncManager.Start()
    // ...
}
```

可以看到，我们终于走到了调用了`syncManager.Start()`的地方。

### `syncManager`中的处理

然后就是在`syncManager`内部的一些处理了。

它主要是除了从`config.toml`中取得种子节点外，还需要把以前连接过并保存在本地的`AddressBook.json`中的节点也拿出来连接，这样就算预设的种子节点失败了，也还是有可能连接上网络（部分解决了前面提到的中心化的担忧）。

`syncManager.Start()`对应于：

[netsync/handle.go#L141](https://github.com/freewind/bytom/blob/master/netsync/handle.go#L141)

```go
func (sm *SyncManager) Start() {
    go sm.netStart()
    // ...
}
```

其中`sm.netStart()`，对应于：

[netsync/handle.go#L121](https://github.com/freewind/bytom/blob/master/netsync/handle.go#L121)

```go
func (sm *SyncManager) netStart() error {
    // ...
    // If seeds exist, add them to the address book and dial out
    if sm.config.P2P.Seeds != "" {
        // dial out
        seeds := strings.Split(sm.config.P2P.Seeds, ",")
        if err := sm.DialSeeds(seeds); err != nil {
            return err
        }
    }
    // ...
}
```

其中的`sm.config.P2P.Seeds`就对应于`config.toml`中的`seeds`。关于这两者是怎么对应起来的，会在后面文章中详解。

紧接着，再通过`sm.DialSeeds(seeds)`去连接这些seed，这个方法对应的代码位于：

[netsync/handle.go#L229](https://github.com/freewind/bytom/blob/master/netsync/handle.go#L229)

```go
func (sm *SyncManager) DialSeeds(seeds []string) error {
    return sm.sw.DialSeeds(sm.addrBook, seeds)
}
```

其实是是调用了`sm.sw.DialSeeds`，而`sm.sw`是指`Switch`。这时可以看到，有一个叫`addrBook`的东西参与了进来，它保存了该结点之前成功连接过的节点地址，我们这里暂不多做讨论。

`Switch.DialSeeds`对应于：

[p2p/switch.go#L311](https://github.com/freewind/bytom/blob/master/p2p/switch.go#L311)

```go
func (sw *Switch) DialSeeds(addrBook *AddrBook, seeds []string) error {
    // ...
    perm := rand.Perm(len(netAddrs))
    for i := 0; i < len(perm)/2; i++ {
        j := perm[i]
        sw.dialSeed(netAddrs[j])
    }
   // ...
}
```

这里引入了随机数，是为了将发起连接的顺序打乱，这样可以让每个种子都获得公平的连接机会。

`sw.dialSeed(netAddrs[j])`对应于：

[p2p/switch.go#L342](https://github.com/freewind/bytom/blob/master/p2p/switch.go#L342)

```go
func (sw *Switch) dialSeed(addr *NetAddress) {
    peer, err := sw.DialPeerWithAddress(addr, false)
    // ...
}
```

`sw.DialPeerWithAddress(addr, false)`又对应于：

[p2p/switch.go#L351](https://github.com/freewind/bytom/blob/master/p2p/switch.go#L351)

```go
func (sw *Switch) DialPeerWithAddress(addr *NetAddress, persistent bool) (*Peer, error) {
    // ...
    log.WithField("address", addr).Info("Dialing peer")
    peer, err := newOutboundPeerWithConfig(addr, sw.reactorsByCh, sw.chDescs, sw.StopPeerForError, sw.nodePrivKey, sw.peerConfig)
    // ...
}
```

其中的`persistent`参数如果是`true`的话，表明这个peer比较重要，在某些情况下如果断开连接后，还会尝试重连。如果`persistent`为`false`的，就没有这个待遇。

`newOutboundPeerWithConfig`对应于：

[p2p/peer.go#L69](https://github.com/freewind/bytom/blob/master/p2p/peer.go#L69)

```go
func newOutboundPeerWithConfig(addr *NetAddress, reactorsByCh map[byte]Reactor, chDescs []*ChannelDescriptor, onPeerError func(*Peer, interface{}), ourNodePrivKey crypto.PrivKeyEd25519, config *PeerConfig) (*Peer, error) {
    conn, err := dial(addr, config)
    // ...
}
```

继续`dial`，加入了超时：

[p2p/peer.go#L284](https://github.com/freewind/bytom/blob/master/p2p/peer.go#L284)

```go
func dial(addr *NetAddress, config *PeerConfig) (net.Conn, error) {
    conn, err := addr.DialTimeout(config.DialTimeout * time.Second)
    if err != nil {
        return nil, err
    }
    return conn, nil
}
```

`addr.DialTimeout`对应于：

[p2p/netaddress.go#L141](https://github.com/freewind/bytom/blob/master/p2p/netaddress.go#L141)

```go
func (na *NetAddress) DialTimeout(timeout time.Duration) (net.Conn, error) {
    conn, err := net.DialTimeout("tcp", na.String(), timeout)
    if err != nil {
        return nil, err
    }
    return conn, nil
}
```

终于到了`net`包的调用，开始真正去连接这个种子节点了，到这里，我们可以认为这个问题解决了。


---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！
