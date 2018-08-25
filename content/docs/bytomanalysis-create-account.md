---
id: bytomanalysis-create-account
title: 通过接口`/create-account`创建帐户
permalink: docs/bytomanalysis-create-account.html
layout: docs
---

在前面，我们探讨了从浏览器的dashboard中进行注册的时候，数据是如何从前端发到后端的，并且后端是如何创建密钥的。而本文将继续讨论，比原是如何通过`/create-account`接口来创建帐户的。

在前面我们知道在`API.buildHandler`中配置了与创建帐户相关的接口配置：

[api/api.go#L164-L244](https://github.com/freewind/bytom-v1.0.1/blob/master/api/api.go#L164-L244)

```go
func (a *API) buildHandler() {
    // ...
    if a.wallet != nil {
        // ...
        m.Handle("/create-account", jsonHandler(a.createAccount))
        // ...
```

可以看到，`/create-account`对应的handler是`a.createAccount`，它是我们本文将研究的重点。外面套着的`jsonHandler`是用来自动JSON与GO数据类型之间的转换的，之前讨论过，这里不再说。

我们先看一下`a.createAccount`的代码：

[api/accounts.go#L15-L30](https://github.com/freewind/bytom-v1.0.1/blob/master/api/accounts.go#L15-L30)

```go
// POST /create-account
func (a *API) createAccount(ctx context.Context, ins struct {
    RootXPubs []chainkd.XPub `json:"root_xpubs"`
    Quorum    int            `json:"quorum"`
    Alias     string         `json:"alias"`
}) Response {

    // 1. 
    acc, err := a.wallet.AccountMgr.Create(ctx, ins.RootXPubs, ins.Quorum, ins.Alias)
    if err != nil {
        return NewErrorResponse(err)
    }

    // 2. 
    annotatedAccount := account.Annotated(acc)
    log.WithField("account ID", annotatedAccount.ID).Info("Created account")

    // 3.
    return NewSuccessResponse(annotatedAccount)
}
```

可以看到，它需要前端传过来`root_xpubs`、`quorum`和`alias`这三个参数，我们在之前的文章中也看到，前端也的确传了过来。这三个参数，通过`jsonHandler`的转换，到这个方法的时候，已经成了合适的GO类型，我们可以直接使用。

这个方法主要分成了三块：

1. 使用`a.wallet.AccountMgr.Create`以及用户发送的参数去创建相应的帐户
2. 调用`account.Annotated(acc)`，把account对象转换成可以被JSON化的对象
3. 向前端发回成功信息。该信息会被jsonHandler自动转为JSON发到前端，用于显示提示信息

第3步没什么好说的，我们主要把目光集中在前两步，下面将依次结合源代码详解。

创建相应的帐户
-------------

创建帐户使用的是`a.wallet.AccountMgr.Create`方法，先看代码：

[account/accounts.go#L145-L174](https://github.com/freewind/bytom-v1.0.1/blob/master/account/accounts.go#L145-L174)

```go
// Create creates a new Account.
func (m *Manager) Create(ctx context.Context, xpubs []chainkd.XPub, quorum int, alias string) (*Account, error) {
    m.accountMu.Lock()
    defer m.accountMu.Unlock()

    // 1.
    normalizedAlias := strings.ToLower(strings.TrimSpace(alias))
    
    // 2.
    if existed := m.db.Get(aliasKey(normalizedAlias)); existed != nil {
        return nil, ErrDuplicateAlias
    }

    // 3. 
    signer, err := signers.Create("account", xpubs, quorum, m.getNextAccountIndex())
    id := signers.IDGenerate()
    if err != nil {
        return nil, errors.Wrap(err)
    }

    // 4.
    account := &Account{Signer: signer, ID: id, Alias: normalizedAlias}
    
    // 5. 
    rawAccount, err := json.Marshal(account)
    if err != nil {
        return nil, ErrMarshalAccount
    }
    
    // 6. 
    storeBatch := m.db.NewBatch()
    accountID := Key(id)
    storeBatch.Set(accountID, rawAccount)
    storeBatch.Set(aliasKey(normalizedAlias), []byte(id))
    storeBatch.Write()

    return account, nil
}
```

我们把该方法分成了6块，这里依次讲解：

1. 把传进来的帐户别名进行标准化修正，比如去掉两头空白并小写
2. 从数据库中寻找该别名是否已经用过。因为帐户和别名是一一对应的，帐户创建成功后，会在数据库中把别名记录下来。所以如果能从数据库中查找，说明已经被占用，会返回一个错误信息。这样前台就可以提醒用户更换。
3. 创建一个`Signer`，实际上就是对`xpubs`、`quorum`等参数的正确性进行检查，没问题的话会把这些信息捆绑在一起，否则返回错误。这个`Signer`我感觉是检查过没问题签个字的意思。
4. 把第3步创建的signer和id，还有前面的标准化之后的别名拿起来，放在一起，就组成了一个帐户
5. 把帐户对象变成JSON，方便后面往数据库里存
6. 把帐户相关的数据保存在数据库，其中别名与id对应（方便以后查询别名是否存在），id与account对象（JSON格式）对应，保存具体的信息

这几步中的第3步中涉及到的方法比较多，需要再细致分析一下：

### signers.Create

[blockchain/signers/signers.go#L67-L90](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/signers/signers.go#L67-L90)

```go
// Create creates and stores a Signer in the database
func Create(signerType string, xpubs []chainkd.XPub, quorum int, keyIndex uint64) (*Signer, error) {
    // 1. 
    if len(xpubs) == 0 {
        return nil, errors.Wrap(ErrNoXPubs)
    }

    // 2.
    sort.Sort(sortKeys(xpubs)) // this transforms the input slice
    for i := 1; i < len(xpubs); i++ {
        if bytes.Equal(xpubs[i][:], xpubs[i-1][:]) {
            return nil, errors.WithDetailf(ErrDupeXPub, "duplicated key=%x", xpubs[i])
        }
    }

    // 3. 
    if quorum == 0 || quorum > len(xpubs) {
        return nil, errors.Wrap(ErrBadQuorum)
    }

    // 4.
    return &Signer{
        Type:     signerType,
        XPubs:    xpubs,
        Quorum:   quorum,
        KeyIndex: keyIndex,
    }, nil
}
```

这个方法可以分成4块，主要就是检查参数是否正确，还是比较清楚的：

1. xpubs不能为空
2. xpubs不能有重复的。检查的时候就先排序，再看相邻的两个是否相等。我觉得这一块代码应该抽出来，比如`findDuplicated`这样的方法，直接放在这里太过于细节了。
3. 检查`quorum`，它是意思是“所需的签名数量”，它必须小于等于xpubs的个数，但不能为0。这个参数到底有什么用这个可能已经触及到比较核心的东西，放在以后研究。
4. 把各信息打包在一起，称之为`Singer`

另外，在第2处还是一个需要注意的`sortKeys`。它实际上对应的是`type sortKeys []chainkd.XPub`，为什么要这么做，而不是直接把`xpubs`传给`sort.Sort`呢？

这是因为，`sort.Sort`需要传进来的对象拥有以下接口：

```go
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}
```

但是`xpubs`是没有的。所以我们把它的类型重新定义成`sortKeys`后，就可以添加上这些方法了：

[blockchain/signers/signers.go#L94-L96](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/signers/signers.go#L94-L96)

```go
func (s sortKeys) Len() int           { return len(s) }
func (s sortKeys) Less(i, j int) bool { return bytes.Compare(s[i][:], s[j][:]) < 0 }
func (s sortKeys) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }
```

### m.getNextAccountIndex()

然后是`signers.Create("account", xpubs, quorum, m.getNextAccountIndex())`中的`m.getNextAccountIndex()`，它的代码如下：

[account/accounts.go#L119-L130](https://github.com/freewind/bytom-v1.0.1/blob/master/account/accounts.go#L119-L130)

```go
func (m *Manager) getNextAccountIndex() uint64 {
    m.accIndexMu.Lock()
    defer m.accIndexMu.Unlock()

    var nextIndex uint64 = 1
    if rawIndexBytes := m.db.Get(accountIndexKey); rawIndexBytes != nil {
        nextIndex = common.BytesToUnit64(rawIndexBytes) + 1
    }

    m.db.Set(accountIndexKey, common.Unit64ToBytes(nextIndex))
    return nextIndex
}
```

从这个方法可以看出，它用于产生自增的数字。这个数字保存在数据库中，其key为`accountIndexKey`（常量，值为`[]byte("AccountIndex")`），value的值第一次为`1`，之后每次调用都会把它加1，返回的同时把它也保存在数据库里。这样比原程序就算重启该数字也不会丢失。

### signers.IDGenerate()

上代码：

[blockchain/signers/idgenerate.go#L21-L41](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/signers/idgenerate.go#L21-L41)

```go
//IDGenerate generate signer unique id
func IDGenerate() string {
    var ourEpochMS uint64 = 1496635208000
    var n uint64

    nowMS := uint64(time.Now().UnixNano() / 1e6)
    seqIndex := uint64(nextSeqID())
    seqID := uint64(seqIndex % 1024)
    shardID := uint64(5)

    n = (nowMS - ourEpochMS) << 23
    n = n | (shardID << 10)
    n = n | seqID

    bin := make([]byte, 8)
    binary.BigEndian.PutUint64(bin, n)
    encodeString := base32.HexEncoding.WithPadding(base32.NoPadding).EncodeToString(bin)

    return encodeString

}
```

从代码中可以看到，这个算法还是相当复杂的，从注释上来看，它是要生成一个“不重复”的id。如果我们细看代码中的算法，发现它没并有和我们的密钥或者帐户有关系，所以我不太明白，如果仅仅是需要一个不重复的id，为什么不能直接使用如uuid这样的算法。另外这个算法是否有名字呢？已经提了issue向开发人员询问：<https://github.com/Bytom/bytom/issues/926>

现在可以回到我们的主线`a.wallet.AccountMgr.Create`上了。关于创建帐户的流程，上面已经基本讲了，但是还有一些地方我们还没有分析：

1. 上面多次提到使用了数据库，那么使用的是什么数据库？在哪里进行了初始化？
3. 这个`a.wallet.AccountMgr.Create`方法中对应的`AccountMgr`对象是在哪里构造出来的？

### 数据库与`AccountMgr`的初始化

比原在内部使用了[leveldb](https://github.com/google/leveldb)这个数据库，从配置文件`config.toml`中就可以看出来：

```ini
$ cat config.toml
fast_sync = true
db_backend = "leveldb"
```

这是一个由Google开发的性能非常高的Key-Value型的NoSql数据库，比特币也用的是它。

比原在代码中使用它保存各种数据，比如区块、帐户等。

我们看一下，它是在哪里进行了初始化。

可以看到，在创建比原节点对象的时候，有大量的与数据库以及帐户相关的初始化操作：

[node/node.go#L59-L142](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L59-L142)

```go
func NewNode(config *cfg.Config) *Node {
    // ...
    
    // Get store
    coreDB := dbm.NewDB("core", config.DBBackend, config.DBDir())
    store := leveldb.NewStore(coreDB)

    tokenDB := dbm.NewDB("accesstoken", config.DBBackend, config.DBDir())
    accessTokens := accesstoken.NewStore(tokenDB)

    // ...

    txFeedDB := dbm.NewDB("txfeeds", config.DBBackend, config.DBDir())
    txFeed = txfeed.NewTracker(txFeedDB, chain)

    // ...

    if !config.Wallet.Disable {
        // 1. 
        walletDB := dbm.NewDB("wallet", config.DBBackend, config.DBDir())
        // 2.
        accounts = account.NewManager(walletDB, chain)
        assets = asset.NewRegistry(walletDB, chain)
        // 3. 
        wallet, err = w.NewWallet(walletDB, accounts, assets, hsm, chain)
        // ...
    }
    // ...
}
```

那么我们在本文中用到的，就是这里的`walletDB`，在上面代码中的数字1对应的地方。

另外，`AccountMgr`的初始化在也这个方法中进行了。可以看到，在第2处，生成的`accounts`对象，就是我们前面提到的`a.wallet.AccountMgr`中的`AccountMgr`。这可以从第3处看到，`accounts`以参数形式传给了`NewWallet`生成了`wallet`对象，它对应的字段就是`AccountMgr`。

然后，当Node对象启动时，它会启动web api服务：

[node/node.go#L169-L180](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L169-L180)

```go
func (n *Node) OnStart() error {
    // ...
    n.initAndstartApiServer()
    // ...
}
```

在`initAndstartApiServer`方法里，又会创建`API`对应的对象：

[node/node.go#L161-L167](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L161-L167)

```go
func (n *Node) initAndstartApiServer() {
    n.api = api.NewAPI(n.syncManager, n.wallet, n.txfeed, n.cpuMiner, n.miningPool, n.chain, n.config, n.accessTokens)
    // ...
}
```

可以看到，它把`n.wallet`对象传给了`NewAPI`，所以`/create-account`对应的handler`a.createAccount`中才可以使用`a.wallet.AccountMgr.Create`，因为这里的`a`指的就是`api`。

这样的话，与创建帐户的流程及相关的对象的初始化我们就都清楚了。

Annotated(acc)
--------------

下面就回到我们的`API.createAccount`中的第2块代码：

```go
    // 2. 
    annotatedAccount := account.Annotated(acc)
    log.WithField("account ID", annotatedAccount.ID).Info("Created account")
```

我们来看一下`account.Annotated(acc)`：

[account/indexer.go#L27-L36](https://github.com/freewind/bytom-v1.0.1/blob/master/account/indexer.go#L27-L36)

```go
//Annotated init an annotated account object
func Annotated(a *Account) *query.AnnotatedAccount {
    return &query.AnnotatedAccount{
        ID:       a.ID,
        Alias:    a.Alias,
        Quorum:   a.Quorum,
        XPubs:    a.XPubs,
        KeyIndex: a.KeyIndex,
    }
}
```

这里出现的`query`指的是比原项目中的一个包`blockchain/query`，相应的`AnnotatedAccount`的定义如下：

[blockchain/query/annotated.go#L57-L63](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/query/annotated.go#L57-L63)

```go
type AnnotatedAccount struct {
    ID       string           `json:"id"`
    Alias    string           `json:"alias,omitempty"`
    XPubs    []chainkd.XPub   `json:"xpubs"`
    Quorum   int              `json:"quorum"`
    KeyIndex uint64           `json:"key_index"`
}
```

可以看到，它的字段与之前我们在创建帐户过程中出现的字段都差不多，不同的是后面多了一些与json相关的注解。在后在前面的`account.Annotated`方法中，也是简单的把`Account`对象里的数字赋值给它。

为什么需要一个`AnnotatedAccount`呢？原因很简单，因为我们需要把这些数据传给前端。在`API.createAccount`的最后，第3步，会向前端返回`NewSuccessResponse(annotatedAccount)`，由于这个值将会被`jsonHandler`转换成JSON，所以它需要有一些跟json相关的注解才行。

同时，我们也可以根据`AnnotatedAccount`的字段来了解，我们最后将会向前端返回什么样的数据。

到这里，我们已经差不多清楚了比原的`/create-account`是如何根据用户提交的参数来创建帐户的。

注：在阅读代码的过程中，对部分代码进行了重构，主要是从一些大方法分解出来了一些更具有描述性的小方法，以及一些变量名称的修改，增加可读性。[#924](https://github.com/Bytom/bytom/pull/924)

---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！