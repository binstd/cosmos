---
id: bytomanalysis-initialization-configuration-file
title: 初始化生成配置文件
permalink: docs/bytomanalysis-initialization-configuration-file.html
layout: docs
---

人们常说，“阅读源代码”是学习编程的一种重要方法。作为程序员，我们在平时的学习工作中，都应该阅读过不少源代码。但是对于大多数人来说，阅读的可能更多是一些代码片断、示例，或者在老师、同事的指导下，先对要阅读的项目代码有了整体的了解之后，再进行针对性的阅读。

但是如果我们面对的是一个像比原这样比较庞大的项目，身边又没有人指导，只能靠自己去看，这时应该怎么来阅读呢？也许每个人也都能找到自己的办法，或高效，或低效，或放弃。

我在这次阅读比原源代码的过程中，尝试的是这样一种方法：从外部入手，通过与比原节点进行数据交互，来一步步了解比原的内部原理。就像剥石榴一样，一点点小心翼翼的下手，最后才能吃到鲜美的果肉。

所以这个文章系列叫作“剥开比原看代码”。

说明
---

在系列中的每一章，我通常都会由一个或者几个相关的问题入手，然后通过对源代码进行分析，来说明比原的代码是如何实现的。对于与当前问题关系不大的代码，则会简单带过，等真正需要它们出场的时候再详细解说。

为了保证文章中引用代码的稳定性，我将基于比原的v1.0.1代码进行分析。随着时间推移，比原的代码也将快速更新，但是我觉得，只要把这个版本的代码理解了，再去看新的代码，应该是一件很容易的事情。

在文章中，将会有一些直接指向github上bytom源代码的链接。为了方便，我专门将bytom v1.0.1的代码放到了一个新的仓库中，这样就不容易与比原官方的最新代码混淆。该仓库地址为：<https://github.com/freewind/bytom-v1.0.1>

当然，你不必clone这个仓库（clone官方仓库<http://github.com/Bytom/bytom>就够了），然后在必要的时候，使用以下命令将代码切换到`v1.0.1`的tag，以便与本系列引用的代码一致：

```
git fetch
git checkout -b v1.0.1
```

不论采用哪种阅读方法，我想第一步都应该先在本地把比原节点跑起来，试试各种功能。

对于如何下载、配置和安装的问题，请直接参看官方文档<https://github.com/Bytom/bytom/tree/v1.0.1>（注意我这里给出的是v1.0.1的文档），这里不多说。

本篇问题
------

当我们本地使用`make bytomd`编译完比原后，我们可以使用下面的命令来进行初始化：

```
./bytomd init --chain_id testnet
```

这里指定了使用的chain是`testnet`（还有别的选项，如`mainnet`等等）。运行成功后，它将会在本地文件系统生成一些配置文件，供比原启动时使用。

所以我的问题是：

比原初始化时，产生了什么样的配置文件，放在了哪个目录下？
----------------------------------------------

下面我将结合源代码，来回答这个问题。

### 目录位置

首先比原在本地会有一个目录专门用于放置各种数据，比如密钥、配置文件、数据库文件等。这个目录对应的代码位于[config/config.go#L190-L205](https://github.com/freewind/bytom-v1.0.1/blob/master/config/config.go#L190-L205)：

```go
func DefaultDataDir() string {
    // Try to place the data folder in the user's home dir
    home := homeDir()
    dataDir := "./.bytom"
    if home != "" {
        switch runtime.GOOS {
        case "darwin":
            dataDir = filepath.Join(home, "Library", "Bytom")
        case "windows":
            dataDir = filepath.Join(home, "AppData", "Roaming", "Bytom")
        default:
            dataDir = filepath.Join(home, ".bytom")
        }
    }
    return dataDir
}
```

可以看到，在不同的操作系统上，数据目录的位置也不同：

1. 苹果系统(`darwin`)：`~/Library/Bytom`
2. Windows(`windows`): `~/AppData/Roaming/Bytom`
3. 其它（如Linux）：`~/.bytom`

### 配置文件内容

我们根据自己的操作系统打开相应的目录（我的是`~/Library/Bytom`），可以看到有一个`config.toml`，内容大约如下：

```ini
$ cat config.toml
# This is a TOML config file.
# For more information, see https://github.com/toml-lang/toml
fast_sync = true
db_backend = "leveldb"
api_addr = "0.0.0.0:9888"
chain_id = "testnet"
[p2p]
laddr = "tcp://0.0.0.0:46656"
seeds = "47.96.42.1:46656,172.104.224.219:46656,45.118.132.164:46656"
```

它已经把一些基本信息告诉我们了，比如：

- `db_backend = "leveldb"`：说明比原内部使用了leveldb作为数据库（用来保存块数据、帐号、交易信息等）
- `api_addr = "0.0.0.0:9888"`：我们可以在浏览器中打开`http://localhost:9888`来访问dashboard页面，进行查看与管理
- `chain_id = "testnet"`：当前连接的是`testnet`，即测试网，里面挖出来的比原币是不值钱的
- `laddr = "tcp://0.0.0.0:46656"`：本地监听`46656`端口，别的节点如果想连我，就需要访问我的`46656`端口
- `seeds = "47.96.42.1:46656,172.104.224.219:46656,45.118.132.164:46656"`：比原启动后，会主动连接这几个地址获取数据

### 内容模板

使用不同的`chain_id`去初始化时，会生成不同内容的配置文件，那么这些内容来自于哪里呢？

原来在[config/toml.go#L22-L45](https://github.com/freewind/bytom/blob/master/config/toml.go#L22-L45)，预定义了不同的模板内容：

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

可以看到，原来这些端口号和seed的地址，都是事先写好在模板里的。

而且，通过观察这些配置，我们可以发现，如果`chain_id`不同，则监听的端口和连接的种子都不同：

1. mainnet（连接到主网）: `46657`，会主动连接6个种子
2. testnet（连接到测试网）: `46656`，会主动连接3个种子
3. solonet（本地单独节点）: `46658`，不会主动连接别人（也因此不会被别人连接上），适合单机研究

### 写入文件

这里我们需要快速的把`bytomd init`的执行流程过一遍，才能清楚配置文件的写入时机，也同时把前面的内容串在了一起。

首先，当我们运行`bytomd init`时，它对应的代码入口为[cmd/bytomd/main.go#L54](https://github.com/freewind/bytom/blob/master/cmd/bytomd/main.go#L54):

```go
func main() {
    cmd := cli.PrepareBaseCmd(commands.RootCmd, "TM", os.ExpandEnv(config.DefaultDataDir()))
    cmd.Execute()
}
```

其中的`config.DefaultDataDir()`就对应于前面提到数据目录位置。

然后执行`cmd.Execute()`，将根据传入的参数`init`，选择下面的函数来执行：[cmd/bytomd/commands/init.go#L25-L24](https://github.com/freewind/bytom/blob/master/cmd/bytomd/commands/init.go#L25-L24)

```go
func initFiles(cmd *cobra.Command, args []string) {
    configFilePath := path.Join(config.RootDir, "config.toml")
    if _, err := os.Stat(configFilePath); !os.IsNotExist(err) {
        log.WithField("config", configFilePath).Info("Already exists config file.")
        return
    }

    if config.ChainID == "mainnet" {
        cfg.EnsureRoot(config.RootDir, "mainnet")
    } else if config.ChainID == "testnet" {
        cfg.EnsureRoot(config.RootDir, "testnet")
    } else {
        cfg.EnsureRoot(config.RootDir, "solonet")
    }

    log.WithField("config", configFilePath).Info("Initialized bytom")
}
```

其中的`configFilePath`，就是`config.toml`的写入地址，即我们前面所说的数据目录下的`config.toml`文件。

`cfg.EnsureRoot`将用来确认数据目录是有效的，并且将根据传入的`chain_id`不同，来生成不同的内容写入到配置文件中。

它对应的代码是[config/toml.go#L10](https://github.com/freewind/bytom/blob/master/config/toml.go#L10)

```go
func EnsureRoot(rootDir string, network string) {
    cmn.EnsureDir(rootDir, 0700)
    cmn.EnsureDir(rootDir+"/data", 0700)

    configFilePath := path.Join(rootDir, "config.toml")

    // Write default config file if missing.
    if !cmn.FileExists(configFilePath) {
        cmn.MustWriteFile(configFilePath, []byte(selectNetwork(network)), 0644)
    }
}
```

可以看到，它对数据目录进行了权限上的确认，并且发现当配置文件存在的时候，不会做任何更改。所以如果我们需要生成新的配置文件，就需要把旧的删除（或改名）。

其中的`selectNetwork(network)`函数，实现了根据`chain_id`的不同来组装不同的配置文件内容，它对应于[master/config/toml.go#L48](https://github.com/freewind/bytom/blob/master/config/toml.go#L48):

```go
func selectNetwork(network string) string {
    if network == "testnet" {
        return defaultConfigTmpl + testNetConfigTmpl
    } else if network == "mainnet" {
        return defaultConfigTmpl + mainNetConfigTmpl
    } else {
        return defaultConfigTmpl + soloNetConfigTmpl
    }
}
```

果然就是一个简单的字符串拼接，其中的`defaultConfigTmpl`和`*NetConfgTmpl`在前面已经出现，这里不重复。

最后调用第三方函数`cmn.MustWriteFile(configFilePath, []byte(selectNetwork(network)), 0644)`，把拼接出来的配置文件内容以权限`0644`写入到指定的文件地址。

到这里，我们这个问题就算回答完毕了。

---

如果你觉得这些文章对你非常有用，控制不住想打赏作者，可以有以下选择：

1. BTM: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`
2. BTC: `1Af2Q23Y1kqgtgbryzjS7RxrnEmyvYuX4b`
3. ETH: `0x6bcCfb7265d4aB0C1a71F7d19b9E581cae73D777`

多少请随意，心意最重要，我们一起努力吧！