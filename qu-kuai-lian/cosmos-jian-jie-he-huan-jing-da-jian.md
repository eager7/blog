# Cosmos简介和环境搭建

\[TOC\]

## 简介

Cosmos是基于Tendermint的多链分布式网络，Tendermint则是基于BFT的共识引擎。 Cosmos解决了目前区块链孤岛的问题，把其他链联合起来构成了一个新的生态系统。 要搞明白cosmos如何解决这个问题，首先需要明白区块链的三层架构：

* 应用层

  负责处理指定的输入（如交易）并更新状态机

* 网络层

  负责广播交易和共识信息

* 共识层

  负责让节点认同当前的系统状态

### 以太坊的缺陷

* 可扩展性

  可扩展性是以太坊的第一个问题，以太坊的TPS只有15，这是因为以太坊还是单链POW模式。

* 可用性

  这里指的是EVM的性能问题，以太坊对开发者的灵活度并不高。因为EVM是一个通用的沙盒系统，并没有针对单一情况进行优化。

* 独立性

  第三个限制是在独立性方面，因为所有的应用都是公用同一个底层环境，当一个DAPP有bug时，除非变更底层应用或者分叉，否则无法修正，或者当一个应用需要新的EVM特性时，也需要对底层环境进行变更。

### Cosmos解决思路

cosmos可以很容易的构建新链，并且让链与链之间方便的通信，相当于创建了一个区块链的网络，通过去中心化的方式进行通信。 通过cosmos，每条链都可以保持独立性，并且可以快速的打包交易。 cosmos使用了一些优秀的开源工具，如tendermint，cosmos SDK，IBC，实际上cosmos就是tendermint团队的社区化产品。

#### Tendermint和ABCI介绍

在过去，构建一条链需要构建链的三个部分：网络层，应用层和共识层。为了让链开发更加简单便捷， Jae Kwon在2014年创建了Tendermint项目。 Tendermint BFT是解决区块链网络层和共识层的通用引擎，可以让区块链的开发者聚焦于应用层的开发，而避免底层复杂的协议。 应用层需要使用socket通过ABCI（Application BlockChain Interface）来调用Tendermint BFT协议。 特性：

* 可以定制私有链和公有链

  Tendermint只实现了网络层和共识层，对于链的具体业务则需要开发者在应用层自行定义，因此可以很方便的开发出公有链或者私有链。

* 高性能

  可实现区块的秒级确认以及数千的TPS

* 实时确认

  BFT算法可以避免分叉，因此Tendermint可以在区块打包后立刻确认交易，实现区块的不可逆化

* 安全机制

  Tendermint不仅有容错机制，同时具有问责机制，问责机制保证侧链共识的安全性。

#### Cosmos SDK和其他应用框架

SDK用来解决应用层通过ABCI和Tendermint连接的问题。

### 区块链互联互通协议IBC

基于Tendermint和cosmos SDK，我们可以很方便的创建一条新链，那么如何将这些链连接在一起呢？ cosmos定义了一个区块链互通协议IBC（Inter-Blockchain Communication protocol），IBC通过使用Tendermint的即时确认机制实现异构链的数据互通。（双向锚定）

#### 什么是异构链

* 架构层不同

  不管是共识层，网络层还是应用层，只要有一层设计不同就可以称为异构链。

  异构链如果想接入IBC，它的共识层必须有较快的确认能力，因此POW是不适合的，它的确认速度太慢。

* 独立主权

  每条链都有自己的出块验证者和资产管理机制。

  **IBC工作模式**

  IBC实际上是通过SPV模式来工作的。

  让我们举一个例子，来看看链A是如何发送10个ATOM代币到链B的。

  首先，每条链都运行一个其他链的轻节点，因此链A和链B都能收到对方的header信息。

  当链A发送IBC Transfer消息后，10个ATOM将会在链A被锁定。

  链A发送一个证明消息到链B，链B收到消息后和链A的header做验证，验证通过，则链B生成10个ATOM。

  **构建区块链网络**

  IBC提供了两条链互通数据的协议，那么怎么构建一个区块链网络呢？

  一个方法是两两直连，但是这种方法会导致连接数非常多，假如一个100条链的网络，就需要建立4950个连接，开销太大

  为了解决这个问题，cosmos设计了两种基础链：Hubs和Zones，Zones是常规的异构区块链，而Hubs则是将区块链连接到一起的链，相当于计算机中的通信总线。

  当Zones和Hubs创建一个IBC连接时，它就相当于连接到了其他链的Zones上了，也就是说其他链只需要建立和Hubs的连接就可以实现区块链网络的建立了。

  创世的Hubs为Cosmos Hubs，它上面的代币称为ATOM。

  **联通非Tendermint共识的链**

  cosmos设计之初就是用来联通所有区块链的，因此它支持快速确认和概率确认两种共识的链。

* 快速确认的链

  快速确认的链通过IBC连接到Hubs上即可。

* 概率确认的链

  像是POW链，就是一种概率确认的链，这种链需要一个快速确认链做为中介来连接到cosmos网络。

  比如以太坊，我们首先需要定义一个不可逆的确认数，假如为100，那么只有确认数达到100时才能执行上面的交易，然后我们定义一个合约，在合约内部定义生效时间为确认数达到100时才执行。

  以太坊的实现已经完成：cosmos/peggy: [https://github.com/cosmos/peggy](https://github.com/cosmos/peggy)

### 解决扩展性

* 垂直扩展

  垂直扩展取决于区块链自身，通过替代POW共识机制以及优化自身架构来提高TPS

* 水平扩展

  水平扩展也就是多链并行的机制，可以大幅度提高总网络的TPS

### 总结

* 通过Tendermint BFT共识和cosmos SDK，cosmos可以方便的开发一条性能强劲的区块链
* 链与链之间通过IBC以及Peg-Zones机制，在保留自身独立性的同时可以方便的进行互通
* 可以快速的进行水平扩展以及垂直扩展

## 环境搭建

### 安装 Gaia

本教程将详细说明如何在你的系统上安装`gaiad`和`gaiacli`。安装后，你可以作为[全节点](/zh/join-mainnet.html)或是[验证人节点](/zh/validators/validator-setup.html)加入到主网。

### 安装 Go

按照[官方文档](https://golang.org/doc/install)安装`go`。记得设置环境变量`$GOPATH`,`$GOBIN`和`$PATH`:

```text
mkdir -p $HOME/go/bin



echo "export GOPATH=$HOME/go" >> ~/.bash_profile



echo "export GOBIN=\$GOPATH/bin" >> ~/.bash_profile



echo "export PATH=\$PATH:\$GOBIN" >> ~/.bash_profile



echo "export GO111MODULE=on" >> ~/.bash_profile



source ~/.bash_profile
```

::: 提示 Cosmos SDK 需要安装 **Go 1.12.1+** :::

### 安装二进制执行程序

接下来，安装最新版本的 Gaia。这里我们使用`master`分支，包含了最新的稳定发布版本。如果需要，请通过`git checkout`命令确定是正确的[发布版本](https://github.com/cosmos/cosmos-sdk/releases)。

::: 警告 对于主网，请确保你的版本大于或等于`v0.33.0` :::

```text
mkdir -p $GOPATH/src/github.com/cosmos



cd $GOPATH/src/github.com/cosmos



git clone https://github.com/cosmos/cosmos-sdk



cd cosmos-sdk && git checkout master



make tools install
```

> _注意_: 如果在这一步中出现问题，请检查你是否安装的是 Go 的最新稳定版本。如果出现下载问题，可以设置go代理：`export GOPROXY=https://goproxy.io`

等`gaiad`和`gaiacli`可执行程序安装完之后，请检查:

```text
$ gaiad version --long



$ gaiacli version --long
```

**注意**：新版本v0.37将gaia拆分成新的库，需要另外安装：`https://github.com/cosmos/gaia`，但是主链运行的还是v0.34.7版本，这个版本需要用go1.12版本编译。

`gaiacli`的返回应该类似于：

```text
cosmos-sdk: 0.33.0



git commit: 7b4104aced52aa5b59a96c28b5ebeea7877fc4f0



go.sum hash: d156153bd5e128fec3868eca9a1397a63a864edb5cfa0ac486db1b574b8eecfe



build tags: netgo ledger



go version go1.12 linux/amd64
```

### Build Tags

build tags 指定了可执行程序具有的特殊特性。

\| Build Tag \| Description \|

\| --------- \| ------------------------------------- \|

\| netgo \| Name resolution will use pure Go code \|

\| ledger \| 支持 Ledger 设备 \(硬件钱包\) \|

### 接下来

然后你可以选择 加入公共测试网 或是 创建私有测试网。

## 加入主网

通过设置config.toml和genesis.json来启动主网`https://hub.cosmos.network/docs/join-mainnet.html` genisis.json在库`https://github.com/cosmos/launch`中，将它下载到本地。 初始化步骤：

```text
➜ config gaiad init --help
Initialize validators's and node's configuration files.


Usage:
  gaiad init [moniker] [flags]


Flags:
      --chain-id string genesis file chain-id, if left blank will be randomly created
  -h, --help help for init
  -o, --overwrite overwrite the genesis.json file


Global Flags:
      --home string directory for config and data (default "/root/.gaiad")
      --inv-check-period uint Assert registered invariants every N blocks
      --log_level string Log level (default "main:info,state:info,*:error")
      --trace print out full stack trace on errors
```

我们可以通过home参数指定配置文件路径，同时需要设置chain-id参数为`cosmoshub-2`。

```text
➜ gaiad init blockabc --chain-id cosmoshub-2 --home /root/cosmos --log_level info
{"app_message":{"auth":{"accounts":[],"params":{"max_memo_characters":"256","sig_verify_cost_ed25519":"590","sig_verify_cost_secp256k1":"1000","tx_sig_limit":"7","tx_size_cost_per_byte":"10"}},"bank":{"send_enabled":true},"crisis":{"constant_fee":{"amount":"1000","denom":"stake"}},"distribution":{"base_proposer_reward":"0.010000000000000000","bonus_proposer_reward":"0.040000000000000000","community_tax":"0.020000000000000000","delegator_starting_infos":[],"delegator_withdraw_infos":[],"fee_pool":{"community_pool":[]},"outstanding_rewards":[],"previous_proposer":"","validator_accumulated_commissions":[],"validator_current_rewards":[],"validator_historical_rewards":[],"validator_slash_events":[],"withdraw_addr_enabled":true},"genutil":{"gentxs":null},"gov":{"deposit_params":{"max_deposit_period":"172800000000000","min_deposit":[{"amount":"10000000","denom":"stake"}]},"deposits":null,"proposals":null,"starting_proposal_id":"1","tally_params":{"quorum":"0.334000000000000000","threshold":"0.500000000000000000","veto":"0.334000000000000000"},"votes":null,"voting_params":{"voting_period":"172800000000000"}},"mint":{"minter":{"annual_provisions":"0.000000000000000000","inflation":"0.130000000000000000"},"params":{"blocks_per_year":"6311520","goal_bonded":"0.670000000000000000","inflation_max":"0.200000000000000000","inflation_min":"0.070000000000000000","inflation_rate_change":"0.130000000000000000","mint_denom":"stake"}},"params":null,"slashing":{"missed_blocks":{},"params":{"downtime_jail_duration":"600000000000","max_evidence_age":"120000000000","min_signed_per_window":"0.500000000000000000","signed_blocks_window":"100","slash_fraction_double_sign":"0.050000000000000000","slash_fraction_downtime":"0.010000000000000000"},"signing_infos":{}},"staking":{"delegations":null,"exported":false,"last_total_power":"0","last_validator_powers":null,"params":{"bond_denom":"stake","max_entries":7,"max_validators":100,"unbonding_time":"1814400000000000"},"redelegations":null,"unbonding_delegations":null,"validators":null},"supply":{"supply":[]}},"chain_id":"cosmoshub-2","gentxs_dir":"","moniker":"blockabc","node_id":"07b42a93606667d04ecb7b81ec8fabd3682878cb"}
```

然后将创世配置文件拷贝到config目录下：

```text
➜ config cp ~/go/src/github.com/cosmos/launch/genesis.json ./
```

启动失败：

```text
➜ config vim config.toml
➜ config gaiad start --home /root/cosmos --log_level info
I[2019-11-07|14:40:08.429] starting ABCI with Tendermint module=main
I[2019-11-07|14:40:08.484] Starting multiAppConn module=proxy impl=multiAppConn
I[2019-11-07|14:40:08.484] Starting localClient module=abci-client connection=query impl=localClient
I[2019-11-07|14:40:08.484] Starting localClient module=abci-client connection=mempool impl=localClient
I[2019-11-07|14:40:08.484] Starting localClient module=abci-client connection=consensus impl=localClient
I[2019-11-07|14:40:08.484] Starting EventBus module=events impl=EventBus
I[2019-11-07|14:40:08.484] Starting PubSub module=pubsub impl=PubSub
I[2019-11-07|14:40:08.488] Starting IndexerService module=txindex impl=IndexerService
I[2019-11-07|14:40:08.488] ABCI Handshake App Info module=consensus height=0 hash= software-version= protocol-version=0
I[2019-11-07|14:40:08.488] ABCI Replay Blocks module=consensus appHeight=0 storeHeight=0 stateHeight=0
panic: stored supply should not have been nil


goroutine 1 [running]:
github.com/cosmos/cosmos-sdk/x/supply/internal/keeper.Keeper.GetSupply(0xc0000f2850, 0x162a140, 0xc0004bb420, 0x163af40, 0xc0000f2fc0, 0x7f893e912c78, 0xc0000f0460, 0xc00094f0e0, 0x1639fc0, 0xc0000340a0, ...)
 /root/go/pkg/mod/github.com/cosmos/cosmos-sdk@v0.34.4-0.20191031200835-02c6c9fafd58/x/supply/internal/keeper/keeper.go:50 +0x18f
```

查询原因，cosmos的主链运行版本为cosmos SDK v0.34.7，因此不能用最新的gaiad版本。。。 用34版本再次启动：

```text
➜ cosmos gaiad start --home /root/cosmos --fast_sync=false --log_level info
I[2019-11-07|15:25:32.664] Starting ABCI with Tendermint module=main
E[2019-11-07|15:25:34.296] Couldn't connect to any seeds module=p2p
```

需要配置p2p种子节点：

```text
➜ cosmos cat config/config.toml
# Comma separated list of seed nodes to connect to
seeds = "3e16af0cead27979e1fc3dac57d03df3c7a77acc@3.87.179.235:26656,ba3bacc714817218562f743178228f23678b2873@public-seed-node.cosmoshub.certus.one:26656,2626942148fd39830cb7a3acccb235fab0332d86@173.212.199.36:26656,3028c6ee9be21f0d34be3e97a59b093e15ec0658@91.205.173.168:26656,89e4b72625c0a13d6f62e3cd9d40bfc444cbfa77@34.65.6.52:26656,6be0856f6365559fdc2e9e97a07d609f754632b0@cosmos-cosmoshub-2-seed.nodes.polychainlabs.com:26656"
```

后台启动：`nohup gaiad start --home /root/cosmos --fast_sync=false --log_level info > cosmos.log 2>&1 &` 通过客户端查询数据：

```text
➜ cosmos gaiacli query block 1 --chain-id cosmoshub-2
{"block_meta":{"block_id":{"hash":"76ADC3B007938986BB552B684F16E027D89993A33127938CA6AEBBF0F50949AA","parts":{"total":"1","hash":"9470D4132A7D9A65540DB32756423106A1CAD84BB381144F958FD9A51E0FC649"}},"header":{"version":{"block":"10","app":"0"},"chain_id":"cosmoshub-2","height":"1","time":"2019-04-22T17:00:00Z","num_txs":"0","total_txs":"0","last_block_id":{"hash":"","parts":{"total":"0","hash":""}},"last_commit_hash":"","data_hash":"","validators_hash":"D658BFD100CA8025CFD3BECFE86194322731D387286FBD26E059115FD5F2BCA0","next_validators_hash":"D658BFD100CA8025CFD3BECFE86194322731D387286FBD26E059115FD5F2BCA0","consensus_hash":"0F2908883A105C793B74495EB7D6DF2EEA479ED7FC9349206A65CB0F9987A0B8","app_hash":"","last_results_hash":"","evidence_hash":"","proposer_address":"B1167D0437DB9DF0D533EE2ACDE48107139BDD2E"}},"block":{"header":{"version":{"block":"10","app":"0"},"chain_id":"cosmoshub-2","height":"1","time":"2019-04-22T17:00:00Z","num_txs":"0","total_txs":"0","last_block_id":{"hash":"","parts":{"total":"0","hash":""}},"last_commit_hash":"","data_hash":"","validators_hash":"D658BFD100CA8025CFD3BECFE86194322731D387286FBD26E059115FD5F2BCA0","next_validators_hash":"D658BFD100CA8025CFD3BECFE86194322731D387286FBD26E059115FD5F2BCA0","consensus_hash":"0F2908883A105C793B74495EB7D6DF2EEA479ED7FC9349206A65CB0F9987A0B8","app_hash":"","last_results_hash":"","evidence_hash":"","proposer_address":"B1167D0437DB9DF0D533EE2ACDE48107139BDD2E"},"data":{"txs":null},"evidence":{"evidence":null},"last_commit":{"block_id":{"hash":"","parts":{"total":"0","hash":""}},"precommits":null}}}
```

通过RPC查询，需要启动cli的后台程序： `nohup gaiacli rest-server --home /root/cosmos --chain-id='cosmoshub-2' --laddr='tcp://0.0.0.0:1317' --trust-node > gaiacli.log 2>&1 &`

## 配置supervisord

```text
➜ supervisord cat cosmos.conf
[program:cosmos]
directory = /root/cosmos ;程序启动目录
command = gaiad start --home /root/cosmos --fast_sync=true --log_level info
autostart = true ; 在 supervisord 启动的时候也自动启动
startsecs = 5 ; 启动 5 秒后没有异常退出，就当作已经正常启动了
autorestart = true ; 程序异常退出后自动重启
startretries = 3 ; 启动失败自动重试次数，默认是 3
user = root ; 用哪个用户启动
redirect_stderr = true ; 把 stderr 重定向到 stdout，默认 false
stdout_logfile_maxbytes = 20MB ; stdout 日志文件大小，默认 50MB
stdout_logfile_backups = 20 ; stdout 日志文件备份数
stdout_logfile = /root/cosmos/log/cosmos.log ;stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录（supervisord 会自动创建日志文件）program:parity]
```

```text
➜ supervisord cat cosmos-rest.conf
[program:cosmos-rest]
directory = /root/cosmos ;程序启动目录
command = gaiacli rest-server --home /root/cosmos --chain-id='cosmoshub-2' --node="tcp://127.0.0.1:26657" --laddr='tcp://0.0.0.0:1317'
autostart = true ; 在 supervisord 启动的时候也自动启动
startsecs = 5 ; 启动 5 秒后没有异常退出，就当作已经正常启动了
autorestart = true ; 程序异常退出后自动重启
startretries = 3 ; 启动失败自动重试次数，默认是 3
user = root ; 用哪个用户启动
redirect_stderr = true ; 把 stderr 重定向到 stdout，默认 false
stdout_logfile_maxbytes = 20MB ; stdout 日志文件大小，默认 50MB
stdout_logfile_backups = 20 ; stdout 日志文件备份数
stdout_logfile = /root/cosmos/log/gaiacli.log ;stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录（supervisord 会自动创建日志文件）program:parity]
```

