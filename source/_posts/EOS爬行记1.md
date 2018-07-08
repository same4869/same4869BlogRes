---
title: EOS爬行记（章一） --- EOS环境搭建与基本命令
date: 2018-7-8
tags: [blockchain]
---
>开篇：虽然币市低迷几近一年，不过并不影响EOS对于大众的关注程度，不管是暴涨还是暴跌，亦或是主网上线还是RAM的热度。作为一个IT从业人员来说，除了买买币之外还有另外的关注了解途径，也可谓是幸运的。笔者几个月下来，从以太坊玩到星云链，到现在也试图跨入EOS的大门，区块链的火爆表象下埋藏的是许许多多的计算机技术，数学知识，经济学原理，正如胡适说的那句话“怕什么真理无穷，进一寸有一寸的欢喜”。
>
>本来也是想网上找找教程就随便玩玩的，没想到EOS的坑实在是太多，相对配套的东西也远远不及以太坊，在这里记录一则是为自己留个备案日后查阅，再则有他人有需要，可以按坑对号查询。

<!-- more -->

### 1.环境说明  
这个很重要，坑多的第一步就是因为不同的系统环境，系统版本，不同的EOS版本等等都会导致各种各样的坑，目前成功的相关参数如下：

- 系统版本：MAC 0S 10.13.4

此外网上有人建议至少20G硬盘与8G以上的内存。
	
### 2.下载源码

```
git clone https://github.com/eosio/eos --recursive
```

最后一个参数需要带上，当然忘了了带上相关的submodule就不会clone下来，执行下一步的时候也会提醒你输入

```
git submodule update --init --recursive
```

来补救。当然，然并卵，不知道是VPN的问题还是其他，总之前几天不管这么clone都下不来，就算把zip下下来了手动考进项目编译也会出现各种各样奇葩的问题，山穷水复中发现了别人下好的全套源码放到百度云里的，抱着破罐子破摔的心理试一试然后就可以了，如果有类似问题的话使用下来链接

```
https://pan.baidu.com/s/1nXq32w2OBwI1GwUTslkBKg
```

### 3.代码编译与安装

进入eos目录，使用

```
./eosio_build.sh
```

然后就是各种下载或者编译进度条，只要不报错就没事，一直等着就行。报错就要搜搜对应的报错。

可以看看参考文档的第二篇，基本有个错误是绝逼会遇到的，不过只要有解决方案就是秒秒钟的事情。

![Alt text](/img/014/20180708-01.jpg)

看到以上这个就证明OK了，喜极而泣。

eos目录中build文件夹已经开放，进去看看里面有啥吧

```
eos/build/programs
```

大概解释如下：

`nodeos：` 区块链服务器节点生成组建

`cleos：` 和区块链交互的接口命令

`keosd：` EOS 钱包

`eosio-launcher：`节点网络组成和部署的应用

在build目录下执行

```
sudo make install
```

可以把一些常用命令配置到系统中，然后可以use anywhere.

#### 4.启动服务器
两条命令

```
cd ~/eos/build/programs/nodeos
./nodeos -e -p eosio --plugin eosio::wallet_api_plugin --plugin eosio::chain_api_plugin --plugin eosio::account_history_api_plugin
```

如果第二条觉得太麻烦可以做如下设置(基本上都会设置)

```
/Users/用户名/Library/Application' 'Support/eosio/nodeos/config

该目录下有一个config.ini文件，配置权限

修改 enable-stale-production = false 为 enable-stale-production = true，记得去掉前面的#

修改 producer-name = eosio ，记得去掉前面的#

添加 plugin = eosio::producer_plugin 

添加 plugin = eosio::wallet_api_plugin 

添加 plugin = eosio::chain_api_plugin 

添加 plugin = eosio::http_plugin

修改完后，下次执行./nodeos即可
```

![Alt text](/img/014/20180708-01.jpg)

看到以上不断输出的数据证明启动成功，CTRL+C可以停止。

#### 5.常用操作与基本命令
单节点跑起来了，后面的操作都是基于这个本地节点的，故都需要重新起一个终端。

看看一些命令

##### 5.1 查看区块信息

```
➜  build git:(master) cleos -p 8888 get info
{
  "server_version": "96ee0325",
  "head_block_num": 16792,
  "last_irreversible_block_num": 16791,
  "head_block_id": "00004198c3c0e383c1e0baf889a06c7c8171ab8d7ce7432ca2c36225e63508ed",
  "head_block_time": "2018-07-08T07:04:44",
  "head_block_producer": "eosio"
}
```

##### 5.2 钱包设置

```
cleos wallet create
```
会创建一个名为default的钱包

##### 5.3 创建其他钱包与查看钱包

```
➜  build git:(master) cleos wallet create --name test
Creating wallet: test
Save password to use in the future to unlock this wallet.
Without password imported keys will not be retrievable.
"PW5JELVZv15CJ51XDTTFBy4ZsEKQ1S3u4XhsANs7huigF8pnwD2bK"
➜  build git:(master) cleos wallet list              
Wallets:
[
  "default *",
  "test *"
]
```

当然命令有错时会给上相应的帮助

```
➜  build git:(master) cleos wallet     
ERROR: RequiredError: Subcommand required
Interact with local wallet
Usage: cleos wallet SUBCOMMAND

Subcommands:
  create                      Create a new wallet locally
  open                        Open an existing wallet
  lock                        Lock wallet
  lock_all                    Lock all unlocked wallets
  unlock                      Unlock wallet
  import                      Import private key into wallet
  list                        List opened wallets, * = unlocked
  keys                        List of private keys from all unlocked wallets in wif format.
```

##### 5.4 为账号部署合约

```
cleos set contract eosio ../../contracts/eosio.bios -p eosio
```

##### 5.5 创建2个key，导入key的私钥
```
cleos create key
Private key: 5Kg4i6WW6mfGXGjriU552KdAQ8tQyCU4mqtevVganWWbzQf1rD1 
Public key: EOS5MSkE5DGgSurc7k3Sv9kWrev6E3GBBqasdiiC3yajPwrW7c4Uq

cleos create key
Private key: 5Kg2P7PRA7wWrW2s53JiaBur7PhDtDsCMZUwQ8Yvn8uAmu8xEMB
Public key: EOS7wERhooVJwqYLuQn5v6UDZnL5KQpGJBDMQoktkxz4baNzicwLX

cleos wallet import 5Kg4i6WW6mfGXGjriU552KdAQ8t QyCU4mqtevVganWWbzQf1rD1
imported private key for: EOS5MSkE5DGgSurc7k3Sv9kWrev6E3GBBqasdiiC3yajPwrW7c 4Uq

cleos wallet import 5Kg2P7PRA7wWrW2s53JiaBur7Ph DtDsCMZUwQ8Yvn8uAmu8xEMB
imported private key for: EOS7wERhooVJwqYLuQn5v6UDZnL5KQpGJBDMQoktkxz4baNzic wLX
```

创建两个账户是因为需要生成两个权限不一样的账户，一个是owner，一个是active。

##### 5.6 􏳔􏲤􏲟􏱼􏰍􏳕􏳓􏰄􏱯􏰘􏳖􏳐根据生成的公钥创建账号
```
cleos create account eosio eostoken EOS5MSkE5
  DGgSurc7k3Sv9kWrev6E3GBBqasdiiC3yajPwrW7c4Uq EOS7wERhooVJwqYLuQn5v6UDZnL5KQp
  GJBDMQoktkxz4baNzicwLX
```

注意这个`eostoken`是账户名，是可以随便取的，不过一旦取了，后面就有很多地方要用，切记。

##### 5.7 查看账户信息
```
➜  build git:(master) cleos get account eostoken
{
  "account_name": "eostoken",
  "permissions": [{
      "perm_name": "active",
      "parent": "owner",
      "required_auth": {
        "threshold": 1,
        "keys": [{
            "key": "EOS6YN37pfrCgUmw8SuWvXo4bCTaJPHm9GaCoKq9hZK7rD6z4uLcT",
            "weight": 1
          }
        ],
        "accounts": []
      }
    },{
      "perm_name": "owner",
      "parent": "",
      "required_auth": {
        "threshold": 1,
        "keys": [{
            "key": "EOS7kkFv5AFj3r83fPmow5CC3bPqeDZXncabiN7at2Dxoz9v3bEvC",
            "weight": 1
          }
        ],
        "accounts": []
      }
    }
  ]
}
```

##### 5.8 检测部署合约
```
cleos get code eostoken
code hash: 0000000000000000000000000000000000000000000000000000000000000000

cleos set contract eostoken ../../contracts/currency

cleos get code eostoken
code hash: d6c891fbdfcff597d82e17c81354574399b01d533e53d412093f03e1950fb9d4

```

##### 5.9 创建货币
```
cleos push action eostoken create '{"issuer":"eostoken","maximum_supply" :"1000000.0000 CUR","can_freeze":"0","can_recall":"0","can_whitelist":"0"}' --permission eostoken@active
```

##### 5.10 发行货币
```
cleos push action eostoken issue '{"to":"eostoken","quantity":"1000.0000
CUR","memo":""}' --permission eostoken@active

```

##### 5.11 查看账号信息
```
cleos get table eostoken eostoken accounts
```

##### 5.12 转账
```
cleos push action eostoken transfer '{"from":"eostoken","to":"eosio","quantity":"20.0000 CUR","memo":"my first transfer"}' --permission eostoken@active
```

#### 6.总结
网上资料不多且鱼龙混杂，想要迈出第一步的确不容易，以上虽然是跑通了但是很多概念很原理都没有系统性的了解，后面的路还很长，后面有新的东西再来更新。

>参考文档：
1.https://blog.csdn.net/munianhua/article/details/79796344
2.http://blog.eosdata.io/index.php/tag/mac/
3.http://www.8btc.com/eos-develop-environment