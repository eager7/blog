# CoinEx 链 Gas 费指南

名词解释：

* **Gas** 为了避免恶意行为，每一笔发送到 CoinEx 链上的交易都需要指定一定数量的 Gas 作为燃料⛽️。CoinEx 链在执行交易时，会根据实际情况（比如交易字节数、签名类型和数量等等）扣除相应的 Gas。提供的 Gas 数不能小于实际执行时消耗的 Gas 数。
* **Gas 价格** 除了 Gas，还需要指定 Gas 的价格。在 CoinEx 链上，Gas 的价格必须是 CET。
* **Gas 费** Gas 乘以 Gas 价格就得到 Gas 费💰：`GasFee = Gas * GasPrice`。Gas 费会作为**节点激励**奖励给出块的**验证者**（Validator）节点。

## 预估 Gas

那么，我们怎么才能知道一笔交易需要多少 Gas 呢？答案是通过 CoinEx 链命令行工具（**CLI**，更多介绍请看[这篇文章](https://forum.coinex.org/t/coinex-chain-testnet-cli/169/2)）提供的 Gas 预估功能。以转账为例，`send`命令（其他交易相关命令也是一样）提供了两个选项来帮助我们预估 Gas：

* `--dry-run` 如果执行命令时提供了这个选项，那么 CLI 不会真正的把交易发送到链上，而只是帮助我们估计 Gas
* `--gas-adjustment` 可以用这个选项调整 Gas 估计值，这个选项的默认值是 1

比如说我们想在测试网 testnet2006 上发送一笔 CET，执行下面的命令就可以帮助我们估计 Gas：

```text
$ ./cetcli tx send cettest1tuj25se6w2x0pd9u8f68tktjd333kuqppva7dc 10000000000cet \
   --from bob \
   --node=47.75.208.217:26657 \
   --chain-id=coinexdex-test2006 \
   --dry-run
gas estimate: 23920
```

可以看到，预估出的 Gas 是 23920。加上`--gas-adjustment`选项再看看：

```text
$ ./cetcli tx send cettest1tuj25se6w2x0pd9u8f68tktjd333kuqppva7dc 10000000000cet \
   --from bob \
   --node=47.75.208.217:26657 \
   --chain-id=coinexdex-test2006 \
   --dry-run \
   --gas-adjustment=1.5
gas estimate: 35880
```

预估出的 Gas 变成了 35880（`23920 * 1.5 = 35880`）。如果我们想保证交易执行成功，或者想让验证者优先打包自己的交易，那么最好是把 Gas 调的稍微高一点。

## 指定 Gas

有了预估的 Gas，就可以通过`--gas`选项来指定，就像下面这样：

```text
./cetcli tx send cettest1tuj25se6w2x0pd9u8f68tktjd333kuqppva7dc 10000000000cet \
  --from bob \
  --node=47.75.208.217:26657 \
  --chain-id=coinexdex-test2006 \
  --gas=35880
```

小提示💡：`--gas`选项的默认值是 200000，所以，如果觉得默认值合理，也可以不指定这个选项。

先别着急执行交易，因为我们还没指定 Gas 费。

## 指定 Gas 费

CoinEx 链 CLI 提供了两种方式来指定 Gas 费：

* 第一：通过`--gas-prices`选项指定 Gas 价格，用 Gas 乘以 Gas 价格就可以算出 Gas 费。
* 第二：直接通过`--fees`选项直接指定 Gas 费。

先来看第一种方式：

```text
$ ./cetcli tx send cettest1tuj25se6w2x0pd9u8f68tktjd333kuqppva7dc 10000000000cet \
  --from bob \
  --node=47.75.208.217:26657 \
  --chain-id=coinexdex-test2006 \
  --gas=35880 \
  --gas-prices=20.0cet
{"chain_id":"coinexdex-test2006","account_number":"46","sequence":"2","fee":{"amount":[{"denom":"cet","amount":"717600"}],"gas":"35880"},"msgs":[{"type":"bankx/MsgSend","value":{"from_address":"cettest1te76rj9p8qrstzyp5mknvekpcsfdr9hjj9mw7j","to_address":"cettest1tuj25se6w2x0pd9u8f68tktjd333kuqppva7dc","amount":[{"denom":"cet","amount":"10000000000"}],"unlock_time":"0"}}],"memo":""}


confirm transaction before signing and broadcasting [y/N]: y
Password to sign with 'bob':
height: 0
txhash: 98062EEABEF44B860B9E10183AF781DDE592836B58501B8F683EED2B2672686C
code: 0
data: ""
rawlog: '[{"msg_index":0,"success":true,"log":"","events":[{"type":"message","attributes":[{"key":"action","value":"send"}]}]}]'
logs:
- msgindex: 0
  success: true
  log: ""
  events:
  - type: message
    attributes:
    - key: action
      value: send
info: ""
gaswanted: 0
gasused: 0
codespace: ""
tx: null
timestamp: ""
events: []
```

可以看到，交易[成功执行](https://testnet.coinex.org/transactions/98062EEABEF44B860B9E10183AF781DDE592836B58501B8F683EED2B2672686C)了，CLI 帮我们算出的 Gas 费是 717600cet（`35880 * 20`）。这里有几点必须要注意：

* 通过`--gas-prices`指定 Gas 价格时，币种只能是`cet`，价格必须是小数（即使是整数也要写成`20.0`这样）
* 指定的金额实际单位是`10^(-8)`CET，换句话说，`20.0cet`实际是`0.00000020CET`
* 目前 Gas 价格不能小于`20.0cet`

所以执行这边交易花的实际 Gas 大约是`0.072CET`。再来看看第二种方式：

```text
$ ./cetcli tx send cettest1tuj25se6w2x0pd9u8f68tktjd333kuqppva7dc 10000000000cet \
   --from bob \
   --node=47.75.208.217:26657 \
   --chain-id=coinexdex-test2006 \
   --gas=35880 \
   --fees=800000cet
{"chain_id":"coinexdex-test2006","account_number":"46","sequence":"3","fee":{"amount":[{"denom":"cet","amount":"800000"}],"gas":"35880"},"msgs":[{"type":"bankx/MsgSend","value":{"from_address":"cettest1te76rj9p8qrstzyp5mknvekpcsfdr9hjj9mw7j","to_address":"cettest1tuj25se6w2x0pd9u8f68tktjd333kuqppva7dc","amount":[{"denom":"cet","amount":"10000000000"}],"unlock_time":"0"}}],"memo":""}


confirm transaction before signing and broadcasting [y/N]: y
Password to sign with 'bob':
height: 0
txhash: E3BCD67DFE2085CD45EB24CC1E56284CE2D6A2E9933E2650A41FC337E1B52176
code: 0
data: ""
rawlog: '[{"msg_index":0,"success":true,"log":"","events":[{"type":"message","attributes":[{"key":"action","value":"send"}]}]}]'
logs:
- msgindex: 0
  success: true
  log: ""
  events:
  - type: message
    attributes:
    - key: action
      value: send
info: ""
gaswanted: 0
gasused: 0
codespace: ""
tx: null
timestamp: ""
events: []
```

可以看到，交易也[执行成功](https://testnet.coinex.org/transactions/E3BCD67DFE2085CD45EB24CC1E56284CE2D6A2E9933E2650A41FC337E1B52176)了，Gas 费正好是`0.08CET`。为了让验证者节点优先打包我们的交易，最好还是多给点 Gas 费😊。有一点需要注意：通过`--fees`选项指定 Gas 费时，金额必须是整数。

小提示💡：如果同时指定`--gas-prices`和`--fees`选项，那么以根据`--gas-prices`选项计算出的 Gas 费为准。

