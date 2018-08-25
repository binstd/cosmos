---
id: bytomanalysis-create-key
title: 通过`/create-key`接口创建密钥
permalink: docs/bytomanalysis-create-key.html
layout: docs
---

在前一篇，我们探讨了从浏览器的dashboard中进行注册的时候，密钥、帐户的别名以及密码，是如何从前端传到了后端。在这一篇，我们就要看一下，当比原后台收到了创建密钥的请求之后，将会如何创建。

由于本文的问题比较具体，所以就不需要再细分，我们直接从代码开始。

还记得在前一篇中，对应创建密钥的web api的功能点的配置是什么样的吗？

在`API.buildHandler`方法中：

[api/api.go#L164-L244](https://github.com/freewind/bytom-v1.0.1/blob/master/api/api.go#L164-L244)

```go
func (a *API) buildHandler() {
    // ...
    if a.wallet != nil {
        // ...
        m.Handle("/create-key", jsonHandler(a.pseudohsmCreateKey))
        // ...
```

可见，其路径为`/create-key`，而相应的handler是`a.pseudohsmCreateKey`（外面套着的`jsonHandler`在之前已经讨论过，这里不提）：

[api/hsm.go#L23-L32](https://github.com/freewind/bytom-v1.0.1/blob/master/api/hsm.go#L23-L32)

```go
func (a *API) pseudohsmCreateKey(ctx context.Context, in struct {
    Alias string `json:"alias"`
    Password string `json:"password"`
}) Response {
    xpub, err := a.wallet.Hsm.XCreate(in.Alias, in.Password)
    if err != nil {
        return NewErrorResponse(err)
    }
    return NewSuccessResponse(xpub)
}
```

它主要是调用了`a.wallet.Hsm.XCreate`，让我们跟进去：

[blockchain/pseudohsm/pseudohsm.go#L50-L66](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/pseudohsm/pseudohsm.go#L50-L66)

```go
// XCreate produces a new random xprv and stores it in the db.
func (h *HSM) XCreate(alias string, auth string) (*XPub, error) {
    // ...
    // 1.
    normalizedAlias := strings.ToLower(strings.TrimSpace(alias))
    // 2.
    if ok := h.cache.hasAlias(normalizedAlias); ok {
        return nil, ErrDuplicateKeyAlias
    }

    // 3.
    xpub, _, err := h.createChainKDKey(auth, normalizedAlias, false)
    if err != nil {
        return nil, err
    }
    // 4.
    h.cache.add(*xpub)
    return xpub, err
}
```

其中出现了`HSM`这个词，它是指`Hardware-Security-Module`，原来比原还预留了跟硬件相关的模块（暂不讨论）。

上面的代码分成了4部分，分别是：

1. 首先对传进来的`alias`参数进行标准化操作，即去两边空白，并且转换成小写
2. 检查`cache`中有没有，有的话就直接返回并报个相应的错，不会重复生成，因为私钥和别名是一一对应的。在前端可以根据这个错误提醒用户检查或者换一个新的别名。
3. 调用`createChainKDKey`生成相应的密钥，并拿到返回的公钥`xpub`
4. 把公钥放入cache中。看起来公钥和别名并不是同一个东西，那前面为什么可以查询alias呢？

所以我们进入`h.cache.hasAlias`看看：

[blockchain/pseudohsm/keycache.go#L76-L84](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/pseudohsm/keycache.go#L76-L84)

```go
func (kc *keyCache) hasAlias(alias string) bool {
    xpubs := kc.keys()
    for _, xpub := range xpubs {
        if xpub.Alias == alias {
            return true
        }
    }
    return false
}
```

通过`xpub.Alias`我们可以了解到，原来别名跟公钥是绑定的，`alias`可以看作是公钥的一个属性（当然也属于相应的私钥）。所以前面把公钥放进cache，之后就可以查询别名了。

那么第3步中的`createChainKDKey`又是如何生成密钥的呢？

[blockchain/pseudohsm/pseudohsm.go#L68-L86](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/pseudohsm/pseudohsm.go#L68-L86)

```go
func (h *HSM) createChainKDKey(auth string, alias string, get bool) (*XPub, bool, error) {
    // 1.
    xprv, xpub, err := chainkd.NewXKeys(nil)
    if err != nil {
        return nil, false, err
    }
    // 2.
    id := uuid.NewRandom()
    key := &XKey{
        ID: id,
        KeyType: "bytom_kd",
        XPub: xpub,
        XPrv: xprv,
        Alias: alias,
    }
    // 3.
    file := h.keyStore.JoinPath(keyFileName(key.ID.String()))
    if err := h.keyStore.StoreKey(file, key, auth); err != nil {
        return nil, false, errors.Wrap(err, "storing keys")
    }
    // 4.
    return &XPub{XPub: xpub, Alias: alias, File: file}, true, nil
}
```

这块代码内容比较清晰，我们可以把它分成4步，分别是：

1. 调用`chainkd.NewXKeys`生成密钥。其中`chainkd`对应的是比原代码库中的另一个包`"crypto/ed25519/chainkd"`，从名称上来看，使用的是`ed25519`算法。如果对前面文章“如何连上一个比原节点”还有印象的话，会记得比原在有新节点连上的时候，就会使用该算法生成一对密钥，用于当次连接进行加密通信。不过需要注意的是，虽然两者都是`ed25519`算法，但是上次使用的代码却是来自第三方库`"github.com/tendermint/go-crypto"`的。它跟这次的算法在细节上究竟有哪些不同，目前还不清楚，留待以后合适的机会研究。然后是传入`chainkd.NewXKeys(nil)`的参数`nil`，对应的是“随机数生成器”。如果传的是`nil`，`NewXKeys`就会在内部使用默认的随机数生成器生成随机数并生成密钥。关于密钥算法相关的内容，在本文中并不探讨。
2. 给当前密钥生成一个唯一的id，在后面用于生成文件名，保存在硬盘上。id使用的是uuid，生成的是一个形如`62bc9340-f6a7-4d16-86f0-4be61920a06e`这样的全球唯一的随机数
3. 把密钥以文件形式保存在硬盘上。这块内容比较多，下面详细讲。
4. 把公钥相关信息组合在一起，供调用者使用。

我们再详细讲一下第3步，把密钥保存成文件。首先是生成文件名，`keyFileName`函数对应的代码如下：

[blockchain/pseudohsm/key.go#L96-L101](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/pseudohsm/key.go#L96-L101)

```go
// keyFileName implements the naming convention for keyfiles:
// UTC--<created_at UTC ISO8601>-<address hex>
func keyFileName(keyAlias string) string {
    ts := time.Now().UTC()
    return fmt.Sprintf("UTC--%s--%s", toISO8601(ts), keyAlias)
}
```

注意这里的参数`keyAlias`实际上应该是`keyID`，就是前面生成的uuid。写成`alias`有点误导，已经提交PR[#922](https://github.com/Bytom/bytom/pull/922)。最后生成的文件名，形如：`UTC--2018-05-07T06-20-46.270917000Z--62bc9340-f6a7-4d16-86f0-4be61920a06e`

生成文件名之后，会通过`h.keyStore.JoinPath`把它放在合适的目录下。通常来说，这个目录是本机数据目录下的`keystore`，如果你是OSX系统，它应该在你的`~/Library/Bytom/keystore`，如果是别的，你可以通过下面的代码来确定[DefaultDataDir()](https://github.com/freewind/bytom-v1.0.1/blob/master/config/config.go#L190-L205)

关于上面的保存密钥文件的目录，到底是怎么确定的，在代码中其实是有点绕的。不过如果你对这感兴趣的话，我相信你应该能自行找到，这里就不列出来了。如果找不到的话，可以试试以下关键字：`pseudohsm.New(config.KeysDir())`, `os.ExpandEnv(config.DefaultDataDir())`, `DefaultDataDir()`，`DefaultBaseConfig()`

在第3步的最后，会调用`keyStore.StoreKey`方法，把它保存成文件。该方法代码如下：

[blockchain/pseudohsm/keystore_passphrase.go#L67-L73](https://github.com/freewind/bytom-v1.0.1/blob/master/blockchain/pseudohsm/keystore_passphrase.go#L67-L73)

```go
func (ks keyStorePassphrase) StoreKey(filename string, key *XKey, auth string) error {
    keyjson, err := EncryptKey(key, auth, ks.scryptN, ks.scryptP)
    if err != nil {
        return err
    }
    return writeKeyFile(filename, keyjson)
}
```

`EncryptKey`里做了很多事情，把传进来的密钥及其它信息利用起来生成了JSON格式的信息，然后通过`writeKeyFile`把它保存硬盘上。所以在你的`keystore`目录下，会看到属于你的密钥文件。它们很重要，千万别误删了。


`a.wallet.Hsm.XCreate`看完了，让我们回到`a.pseudohsmCreateKey`方法的最后一部分。可以看到，当成功生成key之后，会返回一个`NewSuccessResponse(xpub)`，把与公钥相关的信息返回给前端。它会被`jsonHandler`自动转换成JSON格式，通过http返回过去。

在这次的问题中，我们主要研究的是比原在通过web api接口`/create-key`接收到请求后，在内部做了哪些事，以及把密钥文件放在了哪里。其中涉及到密钥的算法（如`ed25519`）会在以后的文章中，进行详细的讨论。


---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！
