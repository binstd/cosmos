---
id: bytomanalysis-dashboard
title: Dashboard开发原理
permalink: docs/bytomanalysis-dashboard.html
layout: docs
---

在前面的几篇文章中，我们一直在研究如何与一个比原节点建立连接，并且从它那里请求区块数据。然而我很快就遇到了瓶颈。

因为当我处理拿到的区块数据时，发现我已经触及到了比原链的核心，即区块链的数据结构以及分叉的处理。如果不能完全理解这一块，就没有办法正确的处理区块数据。然而它涉及的内容太多了，在短时间之内把它理解透彻是一件非常困难的事情。

之前我的做法就好像我想了解一个城市，于是沿着一条路从外围向市中心进发。前面一直很顺利，但等到了市中心时，发现这里人多路杂，有点迷失了。在这种情况下，我觉得我应该暂停研究核心，而是从另外一条路开始，由外向内再来一遍。因为在行进的过程中，我可以慢慢的积累更多的知识，让自己处于学习区而非恐慌区。这条路的终点也将是触及到核心，但是不深入进去。这样的话，等我多走了几条路之后，积累的知识够了，再研究核心就不会觉得迷茫了。

所以本文本来是想去研究一下，当别的节点把区块数据发给我们之后，我们应该怎么处理，现在换成研究比原的Dashboard是怎么做出来的。为什么选择这个呢？因为它非常以一种非常直观的方式，展示了比原向我们提供的各种信息和功能。在本文中，我们并不过多的讲解它上面的功能，而是把关注点放在比原到底是如何在代码层面上实现了这样的一个Dashboard。它上面的功能，将会在以后慢慢研究。

我们今天的问题是“比原的Dashboard是怎么做出来的”，但是这个问题有点大，并且不够具体，所以我们还是跟以前一样，先来把它细分一下：

1. 我们怎样在比原中启用Dashboard功能？
2. Dashboard中提供了哪些信息和功能？
3. 比原是如何实现了http服务器？
4. Dashboard使用了什么样的前端框架？
5. Dashboard上面的数据，是以什么样的方式从后台拿到的？

我们下面开始一一探讨。

我们怎样在比原中启用Dashboard功能？
------------------------------

当我们使用`bytomd node`启动比原节点的时候，不需要任何配置，它就会自动启用Dashboard功能，并且会在浏览器中打开页面，非常方便。

如果是第一次运行，还没有创建过帐户，它会提示我们创建一个帐户及相关的私钥：

![init-account](./images/init-account.png)

我们可以通过填写帐户别名、密钥别名和相应的密码来创建，或者点击下面的"Restore wallet"来恢复之前的帐号（如果之前备份过的话）：

![init-account-filled](./images/init-account-filled.png)

点击"Register"后，就会创建成功，并进入管理页面：

![dashboard1](./images/dashboard1.png)

注意它的地址是：<http://127.0.0.1:9888/dashboard>

如果我们查看配置文件`config.toml`，可以在其中看到它的身影：

```ini
fast_sync = true
db_backend = "leveldb"
api_addr = "0.0.0.0:9888"
chain_id = "solonet"
[p2p]
laddr = "tcp://0.0.0.0:46658"
seeds = ""
```

注意其中的`api_addr`，就是dashboard以及web-api的地址。比原在启动之后，其`BaseConfig.ApiAddress`会从配置文件中取到相应的值：

[config/config.go#L41-L85](https://github.com/freewind/bytom-v1.0.1/blob/master/config/config.go#L41-L85)

```go
type BaseConfig struct {
    // ...
    ApiAddress string `mapstructure:"api_addr"`
    // ...
}
```

然后在启动时，比原的web api以及dashboard会使用该地址，并且在浏览器中打开dashboard。

然而此处有一个奇怪的问题，就是不论这里的值是什么，浏览器总是打开`http://localhost:9888`这个地址。为什么呢？因为它写死在了代码中。

在代码中，`http://localhost:9888`一共出现在了三个地方，一个是用来表示dashboard的访问地址，位于`node/node.go`中：

[node/node.go#L33-L37](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L33-L37)

```go
const (
	webAddress               = "http://127.0.0.1:9888"
	expireReservationsPeriod = time.Second
	maxNewBlockChSize        = 1024
)
```

这里的`webAddress`，只在从代码中打开浏览器显示dashboard时使用：

[node/node.go#L153-L159](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L153-L159)

```go
func lanchWebBroser() {
	log.Info("Launching System Browser with :", webAddress)
	if err := browser.Open(webAddress); err != nil {
		log.Error(err.Error())
		return
	}
}
```

比原通过`"github.com/toqueteos/webbrowser"`这个第三方的库，可以在节点启动的时候，调用系统默认的浏览器，并打开指定的网址，方便了用户。（注意这段代码中有不少错别字，比如`lanch`、`broser`，已在后续版本中修正了）

另一个地方，是用于`bytomcli`这个命令行工具的，只是奇怪的是它放在了`util/util.go`下面：

[util/util.go#L26-L28](https://github.com/freewind/bytom-v1.0.1/blob/master/util/util.go#L26-L28)

```go
var (
	coreURL = env.String("BYTOM_URL", "http://localhost:9888")
)
```

为什么说它是属于`bytomcli`的呢？因为这个`coreURL`最终被用在`util`包下的一个`ClientCall(...)`函数中，用于从代码中向指定的web api发送请求，并使用其回复信息。但是这个方法在`bytomcli`所在的包使用。如果是这样的话，`coreURL`及相关的函数，应该移到`bytomcli`包里才对。

第三个地方，跟第二个非常像，但是位于`tools/sendbulktx/core/util.go`中，它是用于另一个命令行工具`sendbulktx`的：

[tools/sendbulktx/core/util.go#L26-L28](https://github.com/freewind/bytom-v1.0.1/blob/master/tools/sendbulktx/core/util.go#L26-L28)

```go
var (
	coreURL = env.String("BYTOM_URL", "http://localhost:9888")
)
```

一模一样，对吧。其实不光是这里，还有一堆相关的方法和函数，也是一模一样的，一看就是跟第二处互相复制过来的。

关于这里的问题，我提了两个issue：

- [dashboard和web api的地址写在配置文件config.toml中，但是同时写死在代码中](https://github.com/Bytom/bytom/issues/908)：这里在实现上的确是有一定难度的，原因是在配置文件中，写的是`0.0.0.0:9998`，但是从浏览器或者命令行工具中去访问时，需要使用一个具体的ip（而不是`0.0.0.0`），否则某些功能会不正常。另外，在后面的代码分析处会看到，除了配置文件中的这个地址，比原还会优先从环境变量中取得`LISTEN`所对应的地址web api的地址。所以这里需要更多的研究才能正确修复。

- [与读取webapi相关的代码出现大量重复](https://github.com/Bytom/bytom/issues/909)：官方解释说`sendbulktx`这个工具在未来将从bytom项目中独立出去，所以代码是重复的，如果是这样的话，可以接受。

Dashboard中提供了哪些信息和功能？
---------------------------------

下面我们快速过一遍比原的Dashboard提供了哪些信息和功能。由于在本文中，我们关注的重点不是这些具体的功能，所以会不会细究。另外，前面刚创建好的帐号里，很多数据都是没有的，为了展示方便，我事先做了一些数据。

首先是密钥：

![dashboard-keys](./images/dashboard-keys.png)

这里显示了当前有几个密钥，其别名是什么，并且显示出来了主公钥。我们可以点击右上角的“新建”按钮创建多个密钥，但是这里不再展示。

帐户：

![dashboard-account](./images/dashboard-account.png)

资产：

![dashboard-asset](./images/dashboard-asset.png)

默认只定义了`BTM`这一种资产，可以通过“新建”按钮增加多种资产。

余额：

![dashboard-balances](./images/dashboard-balances.png)

看起来我还是相当有钱的（可惜不能用）。

交易：

![dashboard-transactions](./images/dashboard-transactions.png)

展示了多笔交易，实际上是在本机挖矿挖出来的。由于挖矿出来的BTM是由系统直接转到我们的帐户上的，所以也可以看作是一种交易。

创建交易：

![dashboard-create-transaction](./images/dashboard-create-transaction.png)

我们也可以像这样自己创建交易，把我们持有的某种资产（比如BTM）转到另一个地址。

未花费输出：

![dashboard-unspent](./images/dashboard-unspent.png)

简单的理解就是与我相关的每一笔交易都被记录下来，有输入和输出部分，其中的输出可能又是另一个交易的输入。这里显示的是还没有花费掉的输出（可以根据它来计算我当前到底还剩下多少余额）

查看核心状态：

![dashboard-core-status](./images/dashboard-core-status.png)

定义访问控制：

![dashboard-token](./images/dashboard-token.png)

备份和还原操作：

![dashboard-backup](./images/dashboard-backup.png)

另外每个页面左侧栏的下面，还有关于连接的链的类型（此处为`solonet`），以及同步情况和与当前节点连接的其它节点数。

这里展示的信息和功能我们还不需要细究，但是这里出现的名词却是要留意的，因为它们都是比原的核心概念。等我们以后研究比原内部区块链核心功能的时候，实际上都是围绕着它们来的。这里的每一个概念，可能都需要一到多篇文章专门讨论。

我们在今天关注的是技术实现层面，下面我们要开始进入代码时间了。

比原是如何实现了http服务器？
---------------------------------

首先让我们从比原节点启动开始，一直找到启动http服务的地方：

[cmd/bytomd/main.go#L54-L57](https://github.com/freewind/bytom-v1.0.1/blob/master/cmd/bytomd/main.go#L54-L57)

```go
func main() {
	cmd := cli.PrepareBaseCmd(commands.RootCmd, "TM", os.ExpandEnv(config.DefaultDataDir()))
	cmd.Execute()
}
```

[cmd/bytomd/commands/run_node.go#L41-L54](https://github.com/freewind/bytom-v1.0.1/blob/master/cmd/bytomd/commands/run_node.go#L41-L54)

```go
func runNode(cmd *cobra.Command, args []string) error {
	// Create & start node
	n := node.NewNode(config)
	if _, err := n.Start(); err != nil {
    // ..
}
```

[node/node.go#L169-L180](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L169-L180)

```go
func (n *Node) OnStart() error {
	// ...
	n.initAndstartApiServer()
	// ...
}
```

很快找到了，`initAndstartApiServer`：

[node/node.go#L161-L167](https://github.com/freewind/bytom-v1.0.1/blob/master/node/node.go#L161-L167)

```go
func (n *Node) initAndstartApiServer() {
    // 1.
	n.api = api.NewAPI(n.syncManager, n.wallet, n.txfeed, n.cpuMiner, n.miningPool, n.chain, n.config, n.accessTokens)

    // 2. 
	listenAddr := env.String("LISTEN", n.config.ApiAddress)
    env.Parse()
    
    // 3.
	n.api.StartServer(*listenAddr)
}
```

可以看到，该方法分成了三部分：

1. 通过传入大量的参数，来构造一个`API`对象。进去后会看到大量的与url相关的配置。
2. 先从环境中取得`LISTEN`对应的值，如果没有的话，再使用`config.toml`中指定的`api_addr`值，作为api服务的入口地址
3. 真正启动服务

由于2比较简单，所以我们下面将仔细分析1和3.

先找到1处所对应的`api.NewAPI`方法：

[api/api.go#L143-L157](https://github.com/freewind/bytom-v1.0.1/blob/master/api/api.go#L143-L157)

```go
func NewAPI(sync *netsync.SyncManager, wallet *wallet.Wallet, txfeeds *txfeed.Tracker, cpuMiner *cpuminer.CPUMiner, miningPool *miningpool.MiningPool, chain *protocol.Chain, config *cfg.Config, token *accesstoken.CredentialStore) *API {
	api := &API{
		sync:          sync,
		wallet:        wallet,
		chain:         chain,
		accessTokens:  token,
		txFeedTracker: txfeeds,
		cpuMiner:      cpuMiner,
		miningPool:    miningPool,
	}
	api.buildHandler()
	api.initServer(config)

	return api
}
```

它主要就是把传进来的各参数拿住，供后面使用。然后就是`api.buildHandler`来配置各个功能点的路径和处理函数，以及用`api.initServer`来初始化服务。

进入`api.buildHandler()`。这个方法有点长，把它分成几部分来讲解：

[api/api.go#L164-L244](https://github.com/freewind/bytom-v1.0.1/blob/master/api/api.go#L164-L244)

```go
func (a *API) buildHandler() {
    walletEnable := false
    m := http.NewServeMux()
```

看来http服务使用的是Go自带的`http`包。

向下是，当用户的钱包功能没有禁用的话，就会配置与钱包相关的各功能点（比如帐号、交易、密钥等）：

```go
	if a.wallet != nil {
		walletEnable = true

		m.Handle("/create-account", jsonHandler(a.createAccount))
		m.Handle("/list-accounts", jsonHandler(a.listAccounts))
		m.Handle("/delete-account", jsonHandler(a.deleteAccount))

		m.Handle("/create-account-receiver", jsonHandler(a.createAccountReceiver))
		m.Handle("/list-addresses", jsonHandler(a.listAddresses))
		m.Handle("/validate-address", jsonHandler(a.validateAddress))

		m.Handle("/create-asset", jsonHandler(a.createAsset))
		m.Handle("/update-asset-alias", jsonHandler(a.updateAssetAlias))
		m.Handle("/get-asset", jsonHandler(a.getAsset))
		m.Handle("/list-assets", jsonHandler(a.listAssets))

		m.Handle("/create-key", jsonHandler(a.pseudohsmCreateKey))
		m.Handle("/list-keys", jsonHandler(a.pseudohsmListKeys))
		m.Handle("/delete-key", jsonHandler(a.pseudohsmDeleteKey))
		m.Handle("/reset-key-password", jsonHandler(a.pseudohsmResetPassword))

		m.Handle("/build-transaction", jsonHandler(a.build))
		m.Handle("/sign-transaction", jsonHandler(a.pseudohsmSignTemplates))
		m.Handle("/submit-transaction", jsonHandler(a.submit))
		m.Handle("/estimate-transaction-gas", jsonHandler(a.estimateTxGas))

		m.Handle("/get-transaction", jsonHandler(a.getTransaction))
		m.Handle("/list-transactions", jsonHandler(a.listTransactions))

		m.Handle("/list-balances", jsonHandler(a.listBalances))
		m.Handle("/list-unspent-outputs", jsonHandler(a.listUnspentOutputs))

		m.Handle("/backup-wallet", jsonHandler(a.backupWalletImage))
		m.Handle("/restore-wallet", jsonHandler(a.restoreWalletImage))
	} else {
		log.Warn("Please enable wallet")
	}
```

钱包功能默认是启用的，用户如何才能禁用它呢？方法是在配置文件`config.toml`中，加上这一节代码：

```ini
[wallet]
disable = true
```

在前面的代码中，在配置功能点时，使用了大量的`m.Handle("/create-account", jsonHandler(a.createAccount))`这样的代码，它是什么意思呢？

1. `/create-account`：该功能的路径，比如对于这个，用户需要在浏览器或者命令行中，使用地址`http://localhost:9888/create-account`来访问
2. `a.createAccount`：用于处理用户的访问，比如拿到用户提供的数据，处理完后再返回某个数据给用户，会在下面详解
3. `jsonHandler`：是一个中间层，把用户发送的JSON数据转成第2步handler需要的Go类型参数，或者把2返回的Go数据转成JSON给用户
4. `m.Handle(path, handler)`：用来把功能点路径和相应的处理函数对应起来

这里先看第3步中的`jsonHandler`的代码：

[api/api.go#L259-L265](https://github.com/freewind/bytom-v1.0.1/blob/master/api/api.go#L259-L265)

```go
func jsonHandler(f interface{}) http.Handler {
	h, err := httpjson.Handler(f, errorFormatter.Write)
	if err != nil {
		panic(err)
	}
	return h
}
```

它里面用到了`httpjson`，它是比原代码中提供的一个包，位于[net/http/httpjson](https://github.com/freewind/bytom-v1.0.1/blob/master/net/http/httpjson) 。它的功能主要是为了在http访问与Go的函数之间增加了一层转换。通常用户通过http与api交互的时候，发送和接收的都是JSON数据，而我们在第2步的handler中定义的是Go函数，通过`httpjson`，可以在两者之间自动转换，使得我们在写Go代码的时候，不需要考虑JSON以及http协议相关的问题。相应的，为了与jsonhttp配合使用，第2步中的handler在格式上也会有一些要求，详情可参见这里的详细注释：[net/http/httpjson/doc.go#L3-L40](https://github.com/freewind/bytom/blob/f80a6058be22b3888caeb8f327db1f435db43e7a/net/http/httpjson/doc.go#L3-L40) 。由于httpjson所涉及的代码还比较多，这里就不详述，以后有机会专开一篇。

然后我们再看第2步的`a.createAccount`的代码：

[api/accounts.go#L16-L30](https://github.com/freewind/bytom-v1.0.1/blob/master/api/accounts.go#L16-L30)

```go
func (a *API) createAccount(ctx context.Context, ins struct {
	RootXPubs []chainkd.XPub `json:"root_xpubs"`
	Quorum    int            `json:"quorum"`
	Alias     string         `json:"alias"`
}) Response {
	acc, err := a.wallet.AccountMgr.Create(ctx, ins.RootXPubs, ins.Quorum, ins.Alias)
	if err != nil {
		return NewErrorResponse(err)
	}

	annotatedAccount := account.Annotated(acc)
	log.WithField("account ID", annotatedAccount.ID).Info("Created account")

	return NewSuccessResponse(annotatedAccount)
}
```

这个函数的内容我们在这里不细究，需要注意的反而是它的格式，因为前面说了，它需要跟`jsonHandler`配合使用。格式的要求大概就是，第一个参数是`Context`，第二个参数是可以从JSON数据转换过来的参数，返回值是一个Response以及一个Error，但是这四个又全部是可选的。

让我们回到`api.buildHandler()`，继续往下：

```go
	m.Handle("/", alwaysError(errors.New("not Found")))
	m.Handle("/error", jsonHandler(a.walletError))

	m.Handle("/create-access-token", jsonHandler(a.createAccessToken))
	m.Handle("/list-access-tokens", jsonHandler(a.listAccessTokens))
	m.Handle("/delete-access-token", jsonHandler(a.deleteAccessToken))
	m.Handle("/check-access-token", jsonHandler(a.checkAccessToken))

	m.Handle("/create-transaction-feed", jsonHandler(a.createTxFeed))
	m.Handle("/get-transaction-feed", jsonHandler(a.getTxFeed))
	m.Handle("/update-transaction-feed", jsonHandler(a.updateTxFeed))
	m.Handle("/delete-transaction-feed", jsonHandler(a.deleteTxFeed))
	m.Handle("/list-transaction-feeds", jsonHandler(a.listTxFeeds))

	m.Handle("/get-unconfirmed-transaction", jsonHandler(a.getUnconfirmedTx))
	m.Handle("/list-unconfirmed-transactions", jsonHandler(a.listUnconfirmedTxs))

	m.Handle("/get-block-hash", jsonHandler(a.getBestBlockHash))
	m.Handle("/get-block-header", jsonHandler(a.getBlockHeader))
	m.Handle("/get-block", jsonHandler(a.getBlock))
	m.Handle("/get-block-count", jsonHandler(a.getBlockCount))
	m.Handle("/get-difficulty", jsonHandler(a.getDifficulty))
	m.Handle("/get-hash-rate", jsonHandler(a.getHashRate))

	m.Handle("/is-mining", jsonHandler(a.isMining))
	m.Handle("/set-mining", jsonHandler(a.setMining))

	m.Handle("/get-work", jsonHandler(a.getWork))
	m.Handle("/submit-work", jsonHandler(a.submitWork))

	m.Handle("/gas-rate", jsonHandler(a.gasRate))
	m.Handle("/net-info", jsonHandler(a.getNetInfo))
```

可以看到还是各种功能的定义，主要是跟区块数据、挖矿、访问控制等相关的功能，这里就不详述了。

再继续：

```go
	handler := latencyHandler(m, walletEnable)
	handler = maxBytesHandler(handler)
	handler = webAssetsHandler(handler)
	handler = gzip.Handler{Handler: handler}

	a.handler = handler
}
```

这里是把前面定义的功能点配置包成了一个handler，然后在它外面包了一层又一层，添加上了更多的功能：

1. `latencyHandler`：我目前还不能准确说出它的作用，留待以后补充
2. `maxBytesHandler`：防止用户提交的数据过大，目前值约为`10MB`。对于除`signer/sign-block`以外的url有效
3. `webAssetsHandler`：向用户提供dashboard相关的前端页面资源（比如网页、图片等等）。可能是为了性能和方便性方面的考虑，前端文件都经过混淆后，以字符串形式嵌入在[dashboard/dashboard.go](https://github.com/Bytom/bytom/blob/master/dashboard/dashboard.go)中，真正的代码在另一个项目中 <https://github.com/Bytom/dashboard>，我们在后面会看一下
4. `gzip.Handler`：对http客户端进行是否支持`gzip`的检测，并且在支持的情况下，传输数据时使用gzip压缩

然后让我们回到主线，看看前面的`NewAPI`中最后调用的`api.initServer(config)`：

[api/api.go#L89-L122](https://github.com/freewind/bytom-v1.0.1/blob/master/api/api.go#L89-L122)

```go
func (a *API) initServer(config *cfg.Config) {
	// The waitHandler accepts incoming requests, but blocks until its underlying
	// handler is set, when the second phase is complete.
	var coreHandler waitHandler
	var handler http.Handler

	coreHandler.wg.Add(1)
	mux := http.NewServeMux()
	mux.Handle("/", &coreHandler)

	handler = mux
	if config.Auth.Disable == false {
		handler = AuthHandler(handler, a.accessTokens)
	}
	handler = RedirectHandler(handler)

	secureheader.DefaultConfig.PermitClearLoopback = true
	secureheader.DefaultConfig.HTTPSRedirect = false
	secureheader.DefaultConfig.Next = handler

	a.server = &http.Server{
		// Note: we should not set TLSConfig here;
		// we took care of TLS with the listener in maybeUseTLS.
		Handler:      secureheader.DefaultConfig,
		ReadTimeout:  httpReadTimeout,
		WriteTimeout: httpWriteTimeout,
		// Disable HTTP/2 for now until the Go implementation is more stable.
		// https://github.com/golang/go/issues/16450
		// https://github.com/golang/go/issues/17071
		TLSNextProto: map[string]func(*http.Server, *tls.Conn, http.Handler){},
	}

	coreHandler.Set(a)
}
```

这个方法在本文不适合细讲，因为它更多的是涉及到http层面的一些东西，不是本文的重点。值得关注的地方是，方法创建了一个Go提供的`http.Server`，把前面我们辛苦配置好的handler塞进去，万事俱备，只欠启动。

下面就是启动啦。我们终于可以回到最新的`initAndstartApiServer`方法了，还记得它的第3块内容吗？主要就是调用了`n.api.StartServer(*listenAddr)`：

[api/api.go#L125-L140](https://github.com/freewind/bytom-v1.0.1/blob/master/api/api.go#L125-L140)

```go
func (a *API) StartServer(address string) {
	// ...
	listener, err := net.Listen("tcp", address)
	// ...
	go func() {
		if err := a.server.Serve(listener); err != nil {
			log.WithField("error", errors.Wrap(err, "Serve")).Error("Rpc server")
		}
	}()
}
```

这块比较简单，就是使用Go的`net.Listen`来监听传入的web api地址，得到相应的listener之后，把它传给我们在前面创建的`http.Server`的`Serve`方法，就大功告成了。

这一块代码分析写得十分痛苦，主要原因是它的web api这里几乎涉及到了所有比原提供的功能，很庞杂。还有不少跟http协议相关的东西。同时，因为暴露出了接口，这里就容易出现安全风险，所以代码里面还有不少涉及到用户输入、安全检查等。这些东西当然是非常重要的，但是从代码阅读的角度上来讲又难免枯燥，除非我们就是为了研究安全性。

本文的任务主要是研究比原是如何提供http服务的，关于比原在安全性方面做了哪些事情，以后会有专门的分析。

Dashboard使用了什么样的前端框架？
---------------------------------

比原的前端代码是在另一个独立的项目中：https://github.com/Bytom/dashboard

本文我们并不去探讨代码细节，而仅仅去看一下它使用了哪些前端框架，有个大概印象即可。

通过<https://github.com/Bytom/dashboard/blob/master/package.json>我们就可以大概了解到，比原前端使用了：

1. 构建工具：直接利用`npm`的`Scripts`
1. 前端框架：`React` + `Redux`
1. CSS方面：`bootstrap`
1. JavaScript：ES6
1. http请求：`fetch-ponyfill`
1. 资源打包：`webpack`
1. 测试：`mocha`

Dashboard上面的数据，是以什么样的方式从后台拿到的？
----------------------------------------------

以Account相关的代码为例：

[src/sdk/api/accounts.js#L16](https://github.com/Bytom/dashboard/blob/dc80277522bef40315e5a4968f2ed1059261ac36/src/sdk/api/accounts.js#L16)

```JavaScript
const accountsAPI = (client) => {
  return {
    create: (params, cb) => shared.create(client, '/create-account', params, {cb, skipArray: true}),

    createBatch: (params, cb) => shared.createBatch(client, '/create-account', params, {cb}),

    // ...

    listAddresses: (accountId) => shared.query(client, 'accounts', '/list-addresses', {account_id: accountId}),
  }
}
```

这些函数主要是通过`fetch-ponyfill`库中提供的方法，向向前面使用go创建的web api接口发送http请求，并且拿到相应的回复数据。而它们又将在React组件中被调用，拿回来的数据用于填充页面。

同样，更细节的内容在本文就不讲啦。

终于，经过这一大篇的分析，我觉得我对于比原的Dashboard是怎么做出来的，有了一些基本的印象。剩下的，就是在以后，针对其中的功能进行细致的研究。

---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！