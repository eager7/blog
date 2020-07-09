# index

\[TOC\]

## 日志格式

下面是一个标准ERC20日志格式：

```text
        "logs": [


            {



                "address": "0x4f22310c27ef39feaa4a756027896dc382f0b5e2",



                "topics": [



                    "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",



                    "0x000000000000000000000000e71efa3ac6895450da7c154576849d5091a86627",



                    "0x0000000000000000000000009a4f995e5f6471df335191ddc3d3d7326d97b8db"



                ],



                "data": "0x00000000000000000000000000000000000000000000001043561a8829300000",



                "blockNumber": "0x725a68",



                "transactionHash": "0xda27efeaf62a48c36a878041beef7b03b9802c6fdfde0584a852a2487d575169",



                "transactionIndex": "0x88",



                "blockHash": "0xe924f0d6c456b0830d2eb0c4c487891a3dc53d3d19f9bbbd16e2db37f9d23308",



                "logIndex": "0x91",



                "removed": false



            }



        ],
```

下面是一个标准ERC721日志格式：

```text
        "logs": [


            {



                "address": "0x8c9b261faef3b3c2e64ab5e58e04615f8c788099",



                "blockHash": "0xd24d7b86352bf12aab568fe2116998515c481ef056cb93f16346bc79acfcfc19",



                "blockNumber": "0x72e9e1",



                "data": "0x",



                "logIndex": "0x48",



                "removed": false,



                "topics": [



                    "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",



                    "0x000000000000000000000000459a5af4fae179425a0bf9b2c4717f4b214207fc",



                    "0x0000000000000000000000003f873e3d9849254d501bacc89a4804fbf1b24d6a",



                    "0x000000000000000000000000000000000000000000000000000000000000307e"



                ],



                "transactionHash": "0xe75c3a6c0c88a1d62eb15ce653a3bec6c9e5c6a5e16b3ad7df8621a3c3a53df9",



                "transactionIndex": "0x46"



            }



        ],
```

下面是非标准ERC721加密猫的日志格式：

```text
            {


                "address": "0x06012c8cf97bead5deae237070f9587f8e7a266d",



                "blockHash": "0x6b2b7ac0aab1a6ca160cb7f86cfb48152e9a7637f2b239c4c5d2fcf408449439",



                "blockNumber": "0x727694",



                "data": "0x000000000000000000000000b1690c08e213a35ed9bab7b318de14420fb57d8c00000000000000000000000009d47c8ed73c8ff691850291e9c7c44e4a4026db0000000000000000000000000000000000000000000000000000000000160759",



                "logIndex": "0x2a",



                "removed": false,



                "topics": [



                    "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"



                ],



                "transactionHash": "0x3c1629114d025754f70e5107d40e8a4d8dc4932da15df25d7bd4f6031b61dd80",



                "transactionIndex": "0x21"



            }
```

## 日志输出格式如何定义

从上面对比我们发现，有的日志数据是放在topics中，有的放到data中，有的则是topics和data分开存放，那么是什么导致这种区别的呢，我们看看他们事件的定义：

* 0x4f22310c27ef39feaa4a756027896dc382f0b5e2 \(SPIN代币\)

  合约代码：

  ```text
  event Transfer( address indexed from, address indexed to, uint256 value);
  ```

  合约ABI：

  ```text
  {"anonymous":false,"inputs":[{"indexed":true,"name":"from","type":"address"},{"indexed":true,"name":"to","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Transfer","type":"event"}
  ```

* 0x8c9b261faef3b3c2e64ab5e58e04615f8c788099 \(MLBNFT代币\)

  合约代码：

  ```text
  event Transfer( address indexed _from,address indexed _to, uint256 indexed _tokenId);
  ```

  合约ABI：

  ```text
  {"anonymous":false,"inputs":[{"indexed":true,"name":"_from","type":"address"},{"indexed":true,"name":"_to","type":"address"},{"indexed":true,"name":"_tokenId","type":"uint256"}],"name":"Transfer","type":"event"}
  ```

* 0x06012c8cf97bead5deae237070f9587f8e7a266d \(加密猫\)

  合约代码：

  ```text
  event Transfer(address from, address to, uint256 tokenId);
  ```

  合约ABI：

  ```text
  {"anonymous":false,"inputs":[{"indexed":false,"name":"from","type":"address"},{"indexed":false,"name":"to","type":"address"},{"indexed":false,"name":"tokenId","type":"uint256"}],"name":"Transfer","type":"event"}
  ```

  从这几个数据可以看出，数据存放位置和合约定义有关，如果参数带有indexed标记，则数据放在topics中，否则数据存储在data中。

## ABI

ABI 全称是 Application Binary Interface，翻译过来就是：应用程序二进制接口，简单来说就是 以太坊的调用合约时的接口说明。合约的调用和分析在应用端一般通过ABI来发起，否则会特别麻烦，需要手动组装数据。下面是一个ERC20代币的ABI数据： 合约：

```text
contract ERC20 {
    string public constant name = "";    string public constant symbol = "";    uint8 public constant decimals = 0;    function totalSupply() public pure returns (uint);    function balanceOf(address tokenOwner) public pure returns (uint balance);    function allowance(address tokenOwner, address spender) public pure returns (uint remaining);    function transfer(address to, uint tokens) public returns (bool success);    function approve(address spender, uint tokens) public returns (bool success);    function transferFrom(address from, address to, uint tokens) public returns (bool success);    event Transfer(address indexed from, address indexed to, uint tokens);    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);}
```

ABI：

```text
plainchant@plainchant-pct:~/go/src/github.com/eager7/go_study/2019/eth_client/contract/erc20$ solc --abi erc20.sol


======= erc20.sol:ERC20 =======
Contract JSON ABI
[{"constant":true,"inputs":[],"name":"name","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"spender","type":"address"},{"name":"tokens","type":"uint256"}],"name":"approve","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"totalSupply","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"pure","type":"function"},{"constant":false,"inputs":[{"name":"from","type":"address"},{"name":"to","type":"address"},{"name":"tokens","type":"uint256"}],"name":"transferFrom","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"decimals","outputs":[{"name":"","type":"uint8"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[{"name":"tokenOwner","type":"address"}],"name":"balanceOf","outputs":[{"name":"balance","type":"uint256"}],"payable":false,"stateMutability":"pure","type":"function"},{"constant":true,"inputs":[],"name":"symbol","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"to","type":"address"},{"name":"tokens","type":"uint256"}],"name":"transfer","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[{"name":"tokenOwner","type":"address"},{"name":"spender","type":"address"}],"name":"allowance","outputs":[{"name":"remaining","type":"uint256"}],"payable":false,"stateMutability":"pure","type":"function"},{"anonymous":false,"inputs":[{"indexed":true,"name":"from","type":"address"},{"indexed":true,"name":"to","type":"address"},{"indexed":false,"name":"tokens","type":"uint256"}],"name":"Transfer","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"tokenOwner","type":"address"},{"indexed":true,"name":"spender","type":"address"},{"indexed":false,"name":"tokens","type":"uint256"}],"name":"Approval","type":"event"}]
```

ABI定义了函数如果调用，输入输出是什么，定义事件如何获取，参数和存放位置。

## 解析代码

通过上面分析，我们发现很难找到一个通用的方法来解析日志，每个合约的日志情况不一致，导致解析前需要对合约有了解，需要获取到源码或者ABI，但是这两个数据都不会上链，因此无法获取到，我们需要看看还有没有其他方法。

