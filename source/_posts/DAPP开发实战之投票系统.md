---
title: DAPP开发实战之投票系统
date: 2018-4-5
tags: blockchain
---

>&emsp;&emsp;区块链技术其实大概已经出来很长一段时间了，只是在去年随着各种ico着实是火了一把。如同很多刚出现的技术与概念一样，信仰者奉之为神，觉得甚至是可以在短时间内颠覆支付宝的东西；而不信的人又是嗤之以鼻，认为不过都是些骗人的老套路套着新概念又重出江湖了。这是最好的时代，也是最坏的时代，人人都可能相比于几十年前不用那么勤奋努力很多很多年才能获得足够的财富，而就是因为人人都有机会，才让竞争也变得激烈异常。
>
>&emsp;&emsp;作为技术研究者一般都信仰技术无罪，本文也无意谈太多技术之外的八卦，文章的技术核心内容基本都是来自于网上加以整理和实践，没有比模仿更适合了解和掌握一门新技术了。

<!-- more -->

### 一.项目框架概述
  ![Alt text](/img/013/20180406-01.png)
&emsp;&emsp;上图反应出一种互不信任的，去中心化的投票系统架构图，和其他去中心化系统一样，它并没有一个中央服务器来保存数据，每个运行的节点（用户运行的客户端）都保留一份完整的全链路数据，区块链不断增长且不可篡改，每个节点都完全平等。每次数据的变动都是客户端与各自节点实例的交互，然后广播全网络同步的过程，如下图
![Alt text](/img/013/20180406-02.png)

&emsp;&emsp;当然，这是一种比较理想的情况，在移动互联网时代这可能是最急迫需要解决的问题，相信也很少有人能够忍受在手机上运行DAPP像目前在PC上打开以太坊钱包那样可能等上个一整天。所以，区块链社区也已经出现了一些解决方案，例如提供公共区块链节点的Infura, 以及浏览器插件Metamask等。通过这些方案，就不需要花费大量的硬盘、内存和时间去下载并运行完整的区块链节点，同时也可以利用去中心化的优点。当然这并不是本文所要讨论的重点。

&emsp;&emsp;以下是本次应用的架构图
![Alt text](/img/013/20180406-03.png)

&emsp;&emsp;从图中可以看到，网页通过（HTTP上的）远程过程调用（RPC：Remote Procedure Call）与区块链节点进行通信。`web3.js`已经封装了以太坊规定的全部RPC调用，因此利用它就可以与区块链进行交互，而不必手写那些RPC请求包。使用`web3.js`的另一个好处是，你可以使用自己喜欢的前端框架来构建出色的web应用。

&emsp;&emsp;由于获得一个同步的全节点相当耗时，并占用大量磁盘空间。为了能在我们对区块链的兴趣消失之前掌握如何开发一个去中心化应用，本文将使用`ganache`软件来模拟区块链节点，以便快速开发并测试应用，从而可以将注意力集中在去中心化的思想理解与DApp应用逻辑开发方面。

&emsp;&emsp;接下来，我们先将编写一个投票合约，然后编译合约并将其部署到区块链节点 —— `ganache`上。

&emsp;&emsp;最后，我们将分别通过命令行和网页这两种方式，与区块链进行交互。

### 二.使用Node.js进行第一次迭代
#### 2.1 

>&emsp;&emsp;由于各自环境的不同，这里以MAC为准，其他平台可以自行用谷歌百度一下。

由上文可知，首先是安装`gannache`
`sudo npm install -g ganache-cli`
这个软件相当于可以在本地跑一个私链，可以百度先了解一下
>Ganache：以前叫作 TestRPC，它在 TestRPC 和 Truffle 的集成后被重新命名为 Ganache。Ganache 的工作很简单：创建一个虚拟的以太坊区块链，并生成一些我们将在开发过程中用到的虚拟账号。

安装完毕后在控制台输入`ganache-cli`命令，可以看到以下结果
![Alt text](/img/013/20180406-04.png)
这样就相当于一个私链已经存在了，并且`ganache`默认创建了10个测试账号，每个账号里面也会有一些余额。

当然还有一个GUI版本的，有兴趣也可以用这个
`http://truffleframework.com/ganache/#2/_blank`

&emsp;&emsp;以太坊开发目前都是使用`Solidity`语言进行开发，个人感觉如果有面向对象基础的话就不要从最最基本的语法看起，可能还没看完就直接放弃了。下图是投票合约的主要接口
![Alt text](/img/013/20180406-05.png)
基本上，投票合约Voting包含以下内容：

- 构造函数，用来初始化候选人名单。
- 投票方法Vote()，每次执行就将指定的候选人得票数加 1
- 得票查询方法totalVotesFor()，执行后将返回指定候选人的得票数

有几点需要特别指出：

- 合约状态是持久化到区块链上的，因此对合约状态的修改需要消耗以太币。
- 只有在合约部署到区块链的时候，才会调用构造函数，并且只调用一次。
- 与 web 世界里每次部署代码都会覆盖旧代码不同，在区块链上部署的合约是不可改变的，也就是说，如果你更新合约并再次部署，旧的合约仍然会在区块链上存在，并且合约的状态数据也依然存在。新的部署将会创建合约的一个新的实例。

#### 2.2
下面直接看看投票合约代码

```
pragma solidity ^0.4.18;

contract Voting {

  mapping (bytes32 => uint8) public votesReceived;
  bytes32[] public candidateList;

  function Voting(bytes32[] candidateNames) public {
    candidateList = candidateNames;
  }

  function totalVotesFor(bytes32 candidate) view public returns (uint8) {
    require(validCandidate(candidate));
    return votesReceived[candidate];
  }

  function voteForCandidate(bytes32 candidate) public {
    require(validCandidate(candidate));
    votesReceived[candidate]  += 1;
  }

  function validCandidate(bytes32 candidate) view public returns (bool) {
    for(uint i = 0; i < candidateList.length; i++) {
      if (candidateList[i] == candidate) {
        return true;
      }
    }
    return false;
   }
}
```
大概有几个点

- 编译器版本声明
- 合约类声明，构造函数
- 字典，数组
- 函数，断言

稍微看看应该就基本都能看懂了。

我们使用`solc`库来编译合约代码。然后使用`web3js`库，它能够让你通过RPC与区块链进行交互。我们将在node控制台里用这个库编译和部署合约，并与区块链进行交互。
首先，请确保ganache已经在第一个终端窗口中运行：`~$ ganache-cli`。

```
$ node
> Web3 = require('web3')
> web3 = new Web3(new Web3.providers.HttpProvider("http://localhost:8545"));
> web3.eth.accounts
```

要编译合约，首先需要载入`Voting.sol`文件的内容，然后使用编译器（solc）的compile()方法对合约代码进行编译：

```
> code = fs.readFileSync('Voting.sol').toString()
> solc = require('solc')
> compiledCode = solc.compile(code)
```

以上只要没报错就OK，尤其是`compiledCode`,会出现特别多的输出信息。
其中包含两个重要的字段：

- `compiledCode.contracts[':Voting'].bytecode`: 投票合约编译后的字节码，也是要部署到区块链上的代码。
- `compiledCode.contracts[':Voting'].interface`: 投票合约的接口，被称为应用二进制接口（ABI：Application Binary Interface），它声明了合约中包含的接口方法。无论何时需要跟一个合约进行交互，都需要该合约的abi定义。

#### 2.3
合约编译基本已完成，接下来看看怎么部署到区块链上去。
为此，需要先传入合约的abi定义来创建合约对象`VotingContract`，然后利用该对象完成合约在链上的部署和初始化。
命令如下

```
> abiDefinition = JSON.parse(compiledCode.contracts[':Voting'].interface)
> VotingContract = web3.eth.contract(abiDefinition)
> byteCode = compiledCode.contracts[':Voting'].bytecode
> deployedContract = VotingContract.new(['Rama','Nick','Jose'],{data: byteCode, from: web3.eth.accounts[0], gas: 4700000})
> deployedContract.address
> contractInstance = VotingContract.at(deployedContract.address)
```

调用`VotingContract`对象的`new()`方法来将投票合约部署到区块链。`new()`方法参数列表应当与合约的 构造函数要求相一致。对于投票合约而言，`new()`方法的第一个参数是候选人名单。

`new()`方法的最后一个参数用来声明部署选项。先来看一下这个参数的内容：

```
{
  data: byteCode,             //合约字节码
  from: web3.eth.accounts[0], //部署者账户，将从这个账户扣除执行部署交易的开销
  gas: 4700000                //愿意为本次部署最多支付多少油费，单位：Wei
}
```

- data: 这是合约编译后，需要部署到区块链上的合约字节码。
- from: 区块链必须跟踪是谁部署了一个合约。在本例中，我们简单地利用`web3.eth.accounts`返回的第一个账户，作为部署这个合约的账户。在提交交易之前，你必须拥有并解锁这个账户。不过为了方便起见，ganache默认会自动解锁这10个账户。
- gas: 与区块链进行交互需要消耗资金。这笔钱用来付给矿工，因为他们帮你把代码部署到在区块链里。你必须声明愿意花费多少资金让你的代码包含在区块链中，也就是设定`gas`的值。`from`字段声明的账户的余额将会被用来购买 `gas`。`gas`的价格则由区块链网络设定。

#### 2.4
拿到`contractInstance`实例之后可以根据合约定义的接口进行一系列的操作了。

调用合约的`totalVotesFor()`方法来查看某个候选人的得票数。
例如，下面的代码 查看候选人Rama的得票数：
`contractInstance.totalVotesFor.call('Rama')`

```
{ [String: '0'] s: 1, e: 0, c: [ 0 ] }
是数字 0 的科学计数法表示. 
```

调用合约的`voteForCandidate()`方法投票给某个候选人。下面的代码给Rama投了三次票：

`> contractInstance.voteForCandidate('Rama', {from: web3.eth.accounts[0]})`
`> contractInstance.voteForCandidate('Rama', {from: web3.eth.accounts[0]})`
`> contractInstance.voteForCandidate('Rama', {from: web3.eth.accounts[0]})`

现在我们再次查看Rama的得票数：
`>contractInstance.totalVotesFor.call('Rama').toLocaleString()`

```
'3'
```

每执行一次投票，就会产生一次交易，因此`voteForCandidate()`方法将返回一个交易id，作为交易的凭据。比如：`0xdedc7ae544c3dde74ab5a0b07422c5a51b5240603d31074f5b75c0ebc786bf53`。交易id是交易发生的凭据，交易是不可篡改的，因此任何时候可以使用交易id引用或查看交易内容都会得到同样的结果。对于区块链而言，`交易不可篡改`是其核心特性。

#### 2.5
让用户使用命令行显然是非常不友好的，所以接下来尝试使用网页来作为前端交互页面。
![Alt text](/img/013/20180406-06.png)
页面的主要功能如下：

- 列出所有的候选人及其得票数
- 用户在页面中可以输入候选人的名称，然后点击投票按钮，网页中的JS代码将调用投票合约的`voteForCandidate()`方法 —— 和我们nodejs控制台里的流程一样。

先来看看html代码

```
<!DOCTYPE html>
<html>
<head>
 <title>Hello World DApp</title>
 <link href='/lib/gfonts.css' rel='stylesheet' type='text/css'>
 <link href='/lib/bootstrap.min.css' rel='stylesheet' type='text/css'>
</head>
<body class="container">
 <h1>A Simple Hello World Voting Application</h1>
 <div class="table-responsive">
  <table class="table table-bordered">
   <thead>
    <tr>
     <th>Candidate</th>
     <th>Votes</th>
    </tr>
   </thead>
   <tbody>
    <tr>
     <td>Rama</td>
     <td id="candidate-1"></td>
    </tr>
    <tr>
     <td>Nick</td>
     <td id="candidate-2"></td>
    </tr>
    <tr>
     <td>Jose</td>
     <td id="candidate-3"></td>
    </tr>
   </tbody>
  </table>
 </div>
 <input type="text" id="candidate" />
 <a href="#" onclick="voteForCandidate()" class="btn btn-primary">Vote</a>
</body>
<script src="/lib/web3.js"></script>
<script src="/lib/jquery-3.1.1.slim.min.js"></script>
<script src="./index.js"></script>
</html>
```
里面引用了一些基本的JS库，此外还有`web3.js`和`index.js`，为了方便跑代码把这文章的相关代码都传上了git,需者自取。
>https://github.com/same4869/dappDemo

其他的js文件包括`web3.js`都是库文件，没有业务逻辑，而需要我们关注的是`index.js`这个文件，也是我们自己写的js文件。

```
web3 = new Web3(new Web3.providers.HttpProvider("http://192.168.0.6:8545"));
abi = JSON.parse('[{"constant":false,"inputs":[{"name":"candidate","type":"bytes32"}],"name":"totalVotesFor","outputs":[{"name":"","type":"uint8"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"candidate","type":"bytes32"}],"name":"validCandidate","outputs":[{"name":"","type":"bool"}],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"","type":"bytes32"}],"name":"votesReceived","outputs":[{"name":"","type":"uint8"}],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"x","type":"bytes32"}],"name":"bytes32ToString","outputs":[{"name":"","type":"string"}],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"","type":"uint256"}],"name":"candidateList","outputs":[{"name":"","type":"bytes32"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"candidate","type":"bytes32"}],"name":"voteForCandidate","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"contractOwner","outputs":[{"name":"","type":"address"}],"payable":false,"type":"function"},{"inputs":[{"name":"candidateNames","type":"bytes32[]"}],"payable":false,"type":"constructor"}]')
VotingContract = web3.eth.contract(abi);
contractInstance = VotingContract.at('0x1eb79b83e6e8c9ca16355c5a817ad9fa456e1e03');
candidates = {"Rama": "candidate-1", "Nick": "candidate-2", "Jose": "candidate-3"}

function voteForCandidate(candidate) {
 candidateName = $("#candidate").val();
 try {
  contractInstance.voteForCandidate(candidateName, {from: web3.eth.accounts[0]}, function() {
   let div_id = candidates[candidateName];
   $("#"+div_id).html(contractInstance.totalVotesFor.call(candidateName).toString());
  });
 } catch (err) {
 }
}

$(document).ready(function() {
 candidateNames = Object.keys(candidates);
 for (var i = 0; i < candidateNames.length; i++) {
  let name = candidateNames[i];
  let val = contractInstance.totalVotesFor.call(name).toString()
  $("#"+candidates[name]).html(val);
 }
});
```

为了将页面运行起来，需要对JS代码进行一下调整：
节点的RPC API地址
`web3 = new Web3(new Web3.providers.HttpProvider("http://localhost:8545"));`
HttpProvier()对象的构造函数参数是`web3js`库需要链接的以太坊节点RPC API的URL。

当一个合约部署到区块链上时，将获得一个地址，例如`0x329f5c190380ebcf640a90d06eb1db2d68503a53`。 

由于每次部署都会获得一个不同的地址，因此你需要指定它：
`contractInstance = VotingContract.at('0x329f5c190380ebcf640a90d06eb1db2d68503a53')`

在第二个终端中输入以下命令来启动一个简单的Web服务器，以便我们可以在试验环境中的嵌入浏览器中访问页面：

```
~$ cd ~/repo/chapter1
~/repo/chapter1$ python -m SimpleHTTPServer

(python3使用这个，下同) python -m http.server 80
```
Python的`SimpleHTTPServer`模块将启动在8000端口的监听。
现在，在试验环境的嵌入浏览器中点击刷新按钮。如果一切顺利的话，你应该可以看到投票应用的页面了。 当你在文本框中输入候选人姓名，例如Rama，然后点击按钮后，应该会看到候选人Rama的得票数加 1 。

#### 2.6
如果能看到页面，并能够正常投票，第一个基本demo已经能跑起来了。

总结一下，下面是我们到目前为止已经完成的事情：

- 使用`nodejs`,`npm`和`ganache`作为开发环境。
- 开发简单的投票合约，编译并部署到区块链节点上。
- 使用`nodejs`控制台与合约交互。
- 编写网页与合约交互。
- 所有的投票都保存到区块链上，并且不可修改。
- 任何人都可以独立验证每个候选人获得了多少投票。

### 三.使用Truffle进行第二次迭代
#### 3.1
上一章我们已经基于区块链（ganache仿真器）实现了一个投票合约，并且成功通过nodejs控制台和网页实现了与合约的交互。
![Alt text](/img/013/20180406-07.png)

而接下来我们试着做以下这些事

- 使用`Truffle`框架开发投票应用，它可以方便地编译、部署合约。
- 修改已有的投票应用代码，以便适配开发框架。
- 利用Truffle控制台、网页与投票合约进行交互。
- 对投票合约进行扩展，加入通证（`token`）及购买功能。
- 对前端代码进行扩展，通过网页前端购买股票通证，并利用股票通证为候选人投票。

`Truffle`可以先了解下，和各种PHP,Python,JS框架异曲同工，能帮助开发者开发DAPP事半功倍,省去很多手工操作。

首先还是安装
`~$ npm install -g truffle`

`Truffle`提供了众多的项目模版，可以快速搭建一个去中心化应用的骨架代码。下面的代码使用webpack项目模版来创建应用tfapp:

```
~$ mkdir -p ~/repo/tfapp
~$ cd ~/repo/tfapp
~/repo/tfapp$ truffle unbox webpack
```

初始化一个`Truffle`项目时，它会创建运行一个完整的DApp所需的文件和目录。 可以使用ls命令来查看生成的项目结构：

```
~/repo/tfapp$ ls
README.md       contracts       node_modules      test          webpack.config.js   truffle.js
app          migrations       package.json
~/repo/tfapp$ ls app/
index.html javascripts stylesheets
~/repo/tfapp$ ls contracts/
ConvertLib.sol MetaCoin.sol Migrations.sol
~/repo/tfapp$ ls migrations/
1_initial_migration.js 2_deploy_contracts.js
```

前面的章节中主要自己编写了3个文件，分别是`Voting.sol`,`index.html`与`index.js`，现在对这几个文件分别进行处理，以便应用到Truffle生成的应用中。

`Voting.sol`

> 合约文件不需要修改，直接拷贝到 `contracts `目录即可：
```
~/repo/tfapp$ cp ../chapter1/Voting.sol contracts/
~/repo/tfapp$ ls contracts/
Migrations.sol Voting.sol
```


`index.html`

> 先将页面文件拷贝到`app`目录，覆盖Truffle生成的`index.html`：
> 
```
~/repo/tfapp$ cp ../chapter1/index.html app/
```
由于`Truffle`的`webpack`模版在打包JS脚本时，默认使用`app.js`作为打包入口， 因此，我们将页面文件中对`index.js`的引用改为对`app.js`的引用：
`<script src="app.js"></script>`


`app.js`
> 在`Truffle`下，我们需要重写与区块链交互的JS脚本。由于使用webpack打包，因此可以使用`ES2015`语法。
> 
当使用Truffle来编译和部署合约时，框架会将合约的应用接口定义（abi：Application Binary interface）以及部署地址保存到build/contracts目录中同名的json文件中 —— 我们不需要自己记部署地址了！ 例如，Voting.sol的部署信息对应与build/contracts/Voting.json文件。利用这个文件就可以创建投票合约对象：
>
```
import voting_artifacts from '../../build/contracts/Voting.json'
var Voting = contract(voting_artifacts)
```
>
合约对象的`deployed()`方法返回一个`Promise`，其解析值为该合约对象的部署实例代理（真正的实例在链上），利用这个代理可以执行合约的方法：
>
```
Voting.deployed()
  .then(instance => instance.voteForCandidate('Rama')) 
  .then(() => instance.totalVotesFor.call('Rama'))
  .then(votes => console.log('Rama got votes: ', votes))
```

可以根据git下来的结构目标查看对应的文件自行参考与替换。

#### 3.2
`迁移（migration）`目录的内容非常重要。Truffle使用该目录下的迁移脚本来管理应用合约的部署。 我们在之前的操作中，是通过在 node 控制台中调用合约对象的`new()`方法来将投票合约部署到区块链上。有了Truffle，就再也不需要这么做了。

第一个迁移脚本`1_initial_migration.js`的作用是向区块链部署Migrations合约， 这个合约的作用是存储并跟踪已经部署的最新合约。每次运行迁移任务时，Truffle就会向区块链查询获取 已部署好的合约，然后部署新的合约。在部署完成后，这个脚本会更新Migrations合约中的`last_completed_migration`字段指向最新部署的合约。
可以简单地把Migrations合约当成是一个`数据库表`，字段`last_completed_migration`总是保持最新状态。

将迁移脚本`2_deploy_contracts.js`的内容修改为以下内容，以便部署我们的投票合约Voting：

```
var Voting = artifacts.require("./Voting.sol");
module.exports = function(deployer) {
 deployer.deploy(Voting, ['Rama', 'Nick', 'Jose'], {gas: 290000});
};
```

从上面的代码可以看出，Truffle框架将向迁移脚本传入一个部署器对象（`deployer`），调用其`deploy()`方法即可实现指定合约的部署。

`deploy()`方法的第一个参数为要部署合约的编译对象，调用`artifacts.require()`即可直接将合约代码转换为合约编译对象，例如：`artifacts.require('./Voting.sol')` 。

容易理解，Truffle的`artifacts`对象自动调用solidity编译器来编译合约代码文件并返回编译结果对象。

`deploy()`方法的最后一个参数是合约实例化选项对象，可以用来指定部署代码所需的油费 —— 别忘了部署合约也是交易，因此需要烧点油（`gas`）。gas 数量会随着你的合约大小而变化 —— 确切的说，部署一个合约所需的油费取决于编译生成的合约字节码，不同的字节码指令对应不同的开销，累加起来就可以估算出部署费用。

对于投票合约而言，290000个油就足够了 —— 这个价格是我们为部署这个合约愿意承担的最大费用（`gasLimit`），最终的开支可能用不了这么多。当然，如果你的合约很复杂，有可能你愿意买单的这个上限还不够，那么 节点就会返回一个提示，告诉你部署失败，油资不足 —— `Out of gas`

`deploy()`方法的第一个参数和最后一个参数之间，需要按合约构造函数的参数要求依次传入。例如，对于投票合约，我们只需传入一个候选人名单（数组）。

Truffle在执行任务时，将读取当前目录下的配置文件`truffle.js`。通常我们在该配置文件中声明要连接的以太坊节点地址，例如`localhost:8545`：

```
require('babel-register')module.exports = {
 networks: {
  dev: {
   host: 'localhost',
   port: 8545,
   network_id: '*',
   gas: 470000
  }
 }
}
```

你应该会注意到gas选项。这是一个会应用到所有迁移任务的全局变量。当我们调用`deploy()`方法部署一个合约时，如果没有声明愿意承担的油费，那么Truffle就会采用这个值作为该合约的部署油资。

另一个值得指出的是，Truffle支持将合约部署到多个区块链网络，例如开发网络、私有网络、测试网或公网。 在上面的配置文件中，我们仅定义了一个用于开发的网络dev —— 你知道它指向的是ganache模拟器，Truffle 在执行命令时将自动连接到这个网络。

#### 3.3
在Truffle中执行`compile`命令来编译contracts下的所有合约：

```
~/repo/tfapp$ truffle compile
Compiling Migrations.sol...
Compiling Voting.sol...
Writing artifacts to ./build/contracts
```

在Truffle中执行`migrate`命令将编译后的合约部署到链上：

```
~/repo/tfapp$ truffle migrate
Running migration: 1_initial_migration.js
Deploying Migrations...Migrations: 0x3cee101c94f8a06d549334372181bc5a7b3a8bee
Saving successful migration to network...
Saving artifacts...
Running migration: 2_deploy_contracts.js
Deploying Voting...Voting: 0xd24a32f0ee12f5e9d233a2ebab5a53d4d4986203
Saving successful migration to network...
Saving artifacts...

```
以上如果有编译错误可以根据提示解决。

如果由于油费不足而导致部署失败，可以尝试增加`migrations/2_deploy_contracts.js` 里面的 `gas` 值。比如：

```
deployer.deploy(Voting, ['Rama', 'Nick', 'Jose'], {gas: 500000})
```
如果希望自选一个账户来部署合约，而不是使用默认的`accounts[0]`，可以在迁移脚本中使用from 选项指定，例如：

```
deployer.deploy(Voting, ['Rama', 'Nick', 'Jose'], {gas: 500000,from:'0x8cff691c888afe73ffa3965db39be96ba3b34e49'})
```
 
也可以在 `truffle.js` 中指定默认的用来与区块链交互的账户地址：

```
module.exports = {
 networks: {
  dev: {
   host: 'localhost',
   port: 8545,
   network_id: '*',
   gas: 470000,
   from: '0x8cff691c888afe73ffa3965db39be96ba3b34e49'
  }
 }
}
```

#### 3.4
部署顺利的话，现在就可以通过控制台和网页与合约进行交互了。
使用`Truffle控制台`
在第二个终端中输入`truffle console`命令进入控制台：

```
~/repo/tfapp$ truffle console
truffle(development)> Voting.deployed().then(function(contractInstance) {contractInstance.voteForCandidate('Rama').then(function(v) {console.log(v)})})

{ blockHash: '0x7229f668db0ac335cdd0c4c86e0394a35dd471a1095b8fafb52ebd7671433156',blockNumber: 469628,contractAddress: null,
....
....
```

```
truffle(default)> Voting.deployed().then(function(contractInstance) {contractInstance.totalVotesFor.call('Rama').then(function(v) {console.log(v)})})
{ [String: '1'] s: 1, e: 0, c: [ 1] }
```

注意，truffle 的所有调用都会返回promise，这就是为什么每个响应都被包裹在 `then() `函数里的原因。

通过网页交互，首先使用

```
~/repo/tfapp$ webpack
```
在build目录下生成相应的文件，注意webpack的版本（这里推荐2.X），可以用npm进行调整，被这个坑坑了半天时间。

然后进入`build`目录，先建立网页资源文件的符号连接，然后启动web服务器：

```
~/repo/tfapp/build$ ln -s ~/repo/common/lib lib
~/repo/tfapp/build$ ln -s ~/repo/common/fonts fonts
~/repo/tfapp/build$ python -m SimpleHTTPServer
（python3）python -m http.server 8000
```

现在，在浏览器中点击刷新按钮，没问题的话就能看到效果了。

### 四.引入数字代币进行第三次迭代
#### 4.1 
接下来就是比较有意思又比较熟悉的Token了，在以太坊中，一个重要概念就是通证（token），也就是常说的加密数字币，或者代币。
![Alt text](/img/013/20180406-08.png)
通证就是在以太坊上构建的数字资产，可以用它来表示现实世界里的东西，比如黄金，或者是自己的数字资产（就像货币一样）。通证实际上就是智能合约，并没有什么太多的神奇之处。

- 黄金通证：银行可以有1千克的黄金储备，然后发行1千个通证。买100个黄金通证就等于买100克的黄金。
- 公司股票：公司股票可以用以太坊上的代币来表示。通过支付以太，人们可以购买公司股票。
- 游戏币：在一个多玩家游戏中，游戏者可以用以太购买游戏币，并在游戏中进行消费。
- Golem通证：这是一个基于以太坊的真实项目，个人可以通过租售空闲的 CPU来赚取通证。
- 忠诚度积分：商店可以给购物者发行通证作为忠诚度积分，它可以在将来作为现金回收，或是在第三方市场售卖。

在合约中如何实现通证，实际上并没有限制。但是，以太坊有一个叫做`ERC20`的通证标准，该标准还在不断进化中。`ERC20`通证的优点是很容易与其他的符合`ERC20`标准的通证进行交换，同时，也更容易将你的通证集成到其他DApp中。
总的来说，后续将讨论以下内容：

- 学习并掌握新的数据类型，比如结构体（struct），以便在区块链上组织和存储数据
- 理解通证概念并实现投票应用的通证
- 学习使用以太币进行支付，以太币是以太坊区块链平台的数字加密货币。

一提到投票，通常会想起普通的选举，例如，通过投票来选出一个国家的首相或总统。在这种情况下，每个公民都会有一票，可以投给他们支持的候选人。

还有另外一种`加权投票（weighted voting）`，它常常用于公开上市交易的公司。 在这些公司，股东的投票权取决于其持有的股票数量。比如，如果你拥有 10,000 股公司股票，你就有 10,000 个投票权（而不是普通选举中的一票）。

例如，假设有一个叫做Block的上市公司。公司有 3 个空闲职位，分别是总裁、副总裁和部长，以及一组候选人。该公司希望通过股东投票的方式来决定哪个候选人得到哪个职位。获得最高票数的候选人将会成为总裁，然后是副总裁，最后是部长。

针对这个应用场景，我们可以构建一个DApp来发行公司股票，该应用允许任何人购买股票从而成为股东。 股东基于其拥有的股票数为候选人投票。例如，如果你持有10,000 股，你可以一个候选人投 5,000 股， 另一个候选人 3,000 股，第三个候选人 2,000 股。

以下是我们将要在本章实现应用的图示，任何人都可以调用合约的buy()方法来购买公司发行的股票通证，然后就可以调用合约的voteForCandidate()方法为特定的候选人投票：
![Alt text](/img/013/20180406-09.png)

#### 4.2
我们可以按以下思路来实现加权投票应用：

首先初始化一个新的truffle项目，然后修改关键代码文件：

- 投票合约：Voting.sol
- 合约迁移脚本：2_deploy_contracts.js
- 前端代码：index.html、app.js和app.css

在部署合约时初始化参与竞争的候选人名单。我们前面已经知道了如何实现这一点，在迁移脚本`2_deploy_contracs.js`中完成这个任务。

由于投票人需要先持有公司股票。所以，我们还需要在部署合约时初始化公司发行的股票总量。 这些股票就是构成公司的数字资产。在以太坊的世界中，这些数字资产被称为通证`（Token）`。 因此，从现在开始，我们将会把这些股票称为股票通证。

需要指出的是，股票可以看做是一种通证，但是并非所有的以太坊通证都是股票。股票仅仅是我们前一节中提到的通证使用场景的一种。

我们还需要向投票合约中增加一个新的方法，以便任何人都可以购买这些通证。容易理解，投票人给候选人投票时将使用（消耗）这些股票通证。

接下来还需要添加一个方法来查询投票人的信息，以及他们分别给谁投了票、总共持有多少股票通证、 还有多少可用的通证余额等等。

为了跟踪所有这些数据，我们需要使用几个mapping类型的字段，同时还需要引入新的数据类型 `struct（结构体）`来组织投票人信息。

和原来一样，我们使用truffle的`webpack`项目模版来初始化一个新项目， 并从contracts目录下移除无用的合约文件：

```
~$ mkdir -p ~/repo/tkapp
~$ cd ~/repo/tkapp
~/repo/tkapp$ truffle unbox webpack
~/repo/tkapp$ rm contracts/ConvertLib.sol contracts/MetaCoin.sol
```

#### 4.3
新的合约设计如下
![Alt text](/img/013/20180406-10.png)

之前的投票合约仅仅包含两个状态：数组`candidateList`保存候选人名单，字典`votesReceived`跟踪每个候选人获得的投票。

在加权投票合约中，我们需要额外跟踪一些数据：

- 投票人信息：solidity的`结构体（struct）`类型可以将相关数据组织在一起(类似于JAVA BEAN)。用结构体来存储投票人信息非常好。我们将使用一个struct来存储投票人的账户、已经购买的股票通证数量以及给每个候选人投票时所用的股票数量。例如：

```
struct voter {
  address voterAddress; //投票人账户地址
  uint tokensBought;    //投票人持有的股票通证总量
  uint[] tokensUsedPerCandidate; //为每个候选人消耗的股票通证数量
}
```

- 投票人信息字典：使用一个mapping字典来保存所有的投票人信息，键为投票人账户地址，值为投票人信息。 这样给定一个投票人的账户地址，就可以很方面地提取他的相关信息。我们使用voterInfo来表示该字典。 例如：

```
mapping (address => voter) public voterInfo。
```

- 股票通证的相关信息：使用`totalTokens`来保存通证发行总量，`balanceTokens`保存通证余额，`tokenPrice`保存通证的价格。

在部署合约时，除了指定候选人名单，我们还需要声明股票通证发行总量和股票单价。 因此在合约的构造函数中，需要补充声明这些参数。例如：

```
contract Voting{
  function Voting(uint tokens, uint pricePerToken, bytes32[] candidateNames) public {}
}
```

当股东调用`voteForCandidate()`方法投票给特定候选人时，还需要声明其支持力度 —— 用多少股票来支持 这个候选人。因此，我们需要为该方法添加额外的参数以便传入股票通证数量。例如：

```
contract Voting{
 function voteForCandidate(bytes32 candidate, uint votesInTokens) public {}
}
```

任何人都可以调用`buy()`方法来购买公司发行的股票通证，从而成为公司的股东并获得投票权。 你应该已经注意到了该方法的`payable`修饰符。在Sodility合约中，只有声明为`payable`的方法， 才可以接收支付的货币（`msg.value值`）。

```
contract Voting{
  function buy() payable public returns (uint) {
    //使用msg.value来读取用户的支付金额，这要求方法必须具有payable声明
  }
}
```

下面放出所有合约代码，也可以在git目录中查看

```
pragma solidity ^0.4.18; 

contract Voting {

 struct voter {
  address voterAddress;
  uint tokensBought;
  uint[] tokensUsedPerCandidate;
 }

 mapping (address => voter) public voterInfo;

 mapping (bytes32 => uint) public votesReceived;

 bytes32[] public candidateList;

 uint public totalTokens; 
 uint public balanceTokens;
 uint public tokenPrice;

 function Voting(uint tokens, uint pricePerToken, bytes32[] candidateNames) public {
  candidateList = candidateNames;
  totalTokens = tokens;
  balanceTokens = tokens;
  tokenPrice = pricePerToken;
 }

 function buy() payable public returns (uint) {
  uint tokensToBuy = msg.value / tokenPrice;
  require(tokensToBuy <= balanceTokens);
  voterInfo[msg.sender].voterAddress = msg.sender;
  voterInfo[msg.sender].tokensBought += tokensToBuy;
  balanceTokens -= tokensToBuy;
  return tokensToBuy;
 }

 function totalVotesFor(bytes32 candidate) view public returns (uint) {
  return votesReceived[candidate];
 }

 function voteForCandidate(bytes32 candidate, uint votesInTokens) public {
  uint index = indexOfCandidate(candidate);
  require(index != uint(-1));

  if (voterInfo[msg.sender].tokensUsedPerCandidate.length == 0) {
   for(uint i = 0; i < candidateList.length; i++) {
    voterInfo[msg.sender].tokensUsedPerCandidate.push(0);
   }
  }

  uint availableTokens = voterInfo[msg.sender].tokensBought - totalTokensUsed(voterInfo[msg.sender].tokensUsedPerCandidate);
  require (availableTokens >= votesInTokens);

  votesReceived[candidate] += votesInTokens;
  voterInfo[msg.sender].tokensUsedPerCandidate[index] += votesInTokens;
 }

 function totalTokensUsed(uint[] _tokensUsedPerCandidate) private pure returns (uint) {
  uint totalUsedTokens = 0;
  for(uint i = 0; i < _tokensUsedPerCandidate.length; i++) {
   totalUsedTokens += _tokensUsedPerCandidate[i];
  }
  return totalUsedTokens;
 }

 function indexOfCandidate(bytes32 candidate) view public returns (uint) {
  for(uint i = 0; i < candidateList.length; i++) {
   if (candidateList[i] == candidate) {
    return i;
   }
  }
  return uint(-1);
 }

 function tokensSold() view public returns (uint) {
  return totalTokens - balanceTokens;
 }

 function voterDetails(address user) view public returns (uint, uint[]) {
  return (voterInfo[user].tokensBought, voterInfo[user].tokensUsedPerCandidate);
 }

 function transferTo(address account) public {
  account.transfer(this.balance);
 }

 function allCandidates() view public returns (bytes32[]) {
  return candidateList;
 }
}

```

#### 4.4

合约的`buy()`方法用于提供购买股票的接口。注意关键字`payable`，有了它买股票的人才可以付钱给你。 接收钱没有比这个再简单的了！

```
function buy() payable public returns (uint) {
  uint tokensToBuy = msg.value / tokenPrice;   //根据购买金额和通证单价，计算出购买量   
  require(tokensToBuy <= balanceTokens);       //继续执行合约需要确认合约的通证余额不小于购买量  
  voterInfo[msg.sender].voterAddress = msg.sender;    //保存购买人地址
  voterInfo[msg.sender].tokensBought += tokensToBuy;  //更新购买人持股数量
  balanceTokens -= tokensToBuy;                //将售出的通证数量从合约的余额中剔除  
  return tokensToBuy;                          //返回本次购买的通证数量
}
```

当用户（或程序）调用合约的`buy()`方法时，需要在请求消息里利用`value`属性设置用于购买股票通证的以太币金额。例如：

```
contract.buy({
  value:web3.toWei('1','ether'), //购买者支付的以太币金额
  from:web3.eth.accounts[1]      //购买者账户地址
})
```

在合约的`payable`方法实现代码中使用`msg.value`来读取用户支付的以太币数额。 基于用户支付额和股票通证单价，就可以计算出购买数量，并将这些通证赋予购买人， 购买人的账户地址可以通过`msg.sender`获取。

当然，也可以从truffle控制台调用`buy()`方法来购买股票通证：

```
truffle(development)> Voting.deployed().then(function(contract) {contract.buy({value: web3.toWei('1', 'ether'), from: web3.eth.accounts[1]})})
```

如前所述，加权投票方法不仅要指定候选人名称，还要指定使用多少股票通证来支持该候选人。 我们分别用`candidate`和`votesInTokens`来表示这两个参数：

```
function voteForCandidate(bytes32 candidate, uint votesInTokens) public {}
```

在投票人调用`voteForCandidate()`方法投票时，我们不仅需要为指定的候选人增加其投票数，还需要跟踪投票人的相关信息，比如投票人是谁（即其账户地址），以及给每个候选人投了多少票。因此在该方法的开始部分，检查如果是该投票人第一次参与投票的话，首先初始化该投票人的`voterInfo`结构：

```
if (voterInfo[msg.sender].tokensUsedPerCandidate.length == 0) {
 for(uint i = 0; i < candidateList.length; i++) { 
  voterInfo[msg.sender].tokensUsedPerCandidate.push(0);  //该投票人为每个候选人投入的通证数量初始化为0
 }
}
```

接下来我们计算该投票人当前的有效持股数量 —— 从该投票人的持股数量中扣除其为所有投票人已经消耗的股票通证数量：

```
uint availableTokens = voterInfo[msg.sender].tokensBought -  totalTokensUsed(voterInfo[msg.sender].tokensUsedPerCandidate)
```

显然，在合约继续执行之前，需要满足条件 —— 投票人的有效持股数量不小于本次投票使用的股票通证数量：

```
require (availableTokens >= votesInTokens)
```

如果投票人依然持有足够数量的股票通证，我们就更新候选人获得的票数，同时更新投票人的通证使用记录：

```
votesReceived[candidate] += votesInTokens;
voterInfo[msg.sender].tokensUsedPerCandidate[index] += votesInTokens;
```

当一个用户调用`buy()`方法发送以太来购买了合约发行的股票通证后，合约收到的资金去了哪里？

所有收到的资金（以太）都在这个投票合约里。每个合约都有它自己的地址，因此也是一个账户。 在以太坊里，这种账户被称为`合约账户（Contract Account）`，而之前的人员账户，则被称为`外控账户 (External Controlled Account)`。
因此，合约的地址里存着这些销售收益。

我们新增加的`transferTo()`方法，可以将合约里的资金转移到指定账户：

```
function transferTo(address account) public {
  account.transfer(this.balance);
}
```

注意！`transferTo()`方法的当前实现，并没有限制调用者，因此任何人都可以调用该方法从而转走投票合约账户里的资金！在生产系统中，你必须添加一些限制条件来避免上面的资金漏洞，例如，检查目标账户是否在一个白名单里。

合约里面剩下的方法都是辅助性的`getter`方法，仅仅用来返回合约变量的值。

注意`tokensSold()`等方法声明中的`constant`修饰符，这表明该方法是只读的，即方法的执行 并不会改变区块链的状态，因此执行这些交易不会耗费任何gas。

#### 4.5

与之前类似，我们修改迁移脚本`2_deploy_contracts.js`来自动化投票合约的部署。 不过由于新的加权投票合约的构造函数声明了额外的参数，因此需要在迁移脚本中传入两个额外的参数 ：

```
var Voting = artifacts.require("./Voting.sol");
module.exports = function(deployer) {
 deployer.deploy(Voting, 10000, web3.toWei('0.01', 'ether'), ['Rama', 'Nick', 'Jose']);
};
```

在上面的代码中，我们部署的合约发行了10000个股票通证，单价为0.01以太。由于所有的价格需要以`Wei`为单位计价，所以我们需要用`toWei()`方法将Ether转换为Wei。

>以太币面值
>
> Wei 是 Ether 的最小面值。1 Ether 等于 1000000000000000000 Wei —— 18个0。 
> 
> 你可以把它当成是美分与美元，就像 Nickel（5 美分），Dime（10 美分），Quarter（25 美分），Ether 也有不同面值。其他面值如下：
> 
- kwei/babbage
- mwei/lovelace
- gwei/shannon
- szabo
- finney
- ether
- kether/grand/einstein
- mether
- gether
- tether

当然也可以在 truffle 控制台，执行 `web3.toWei(1, 'ether')` 来看一下ether（或其他面值）与 wei 之间 的转换关系。例如：

```
truffle(development)> web3.toWei(1,'ether')
```

现在可以编译合约并将其部署到区块链了：

```
~/repo/tkapp$ truffle compile
Compiling Migrations.sol...
Compiling Voting.sol...
Writing artifacts to ./build/contracts

~/repo/tkapp$ truffle migrate
Running migration: 1_initial_migration.js
Deploying Migrations...Migrations: 0x3cee101c94f8a06d549334372181bc5a7b3a8bee
Saving successful migration to network...
Saving artifacts...
Running migration: 2_deploy_contracts.js
Deploying Voting...Voting: 0xd24a32f0ee12f5e9d233a2ebab5a53d4d4986203
Saving successful migration to network...
Saving artifacts...

```

#### 4.6
成功地将合约部署到了`ganache`后，执行`truffle console`进入控制台，让我们和合约互动一下：

- 一个候选人（比如 Nick）有多少投票？

```
truffle(development)> Voting.deployed().then(function(instance) {instance.totalVotesFor.call('Nick').then(function(i) {console.log(i)})})
```

- 一共初始化发行了多少通证？

```
truffle(development)> Voting.deployed().then(function(instance) {console.log(instance.totalTokens().then(function(v) {console.log(v)}))})
```

- 已经售出了多少通证？

```
truffle(development)> Voting.deployed().then(function(instance) {console.log(instance.tokensSold().then(function(v) {console.log(v)}))})
```

- 购买 100个通证

```
truffle(development)> Voting.deployed().then(function(instance) {console.log(instance.buy({value: web3.toWei('1', 'ether')}).then(function(v) {console.log(v)}))})
```

- 购买以后账户余额是多少？

```
truffle(development)> web3.eth.getBalance(web3.eth.accounts[0])
```

- 已经售出了多少通证？

```
Voting.deployed().then(function(instance) {console.log(instance.tokensSold().then(function(v) {console.log(v)}))})
```

- 给 Jose 投 25 个 通证，给 Rama 和 Nick 各投 10 个 通证。

```
truffle(development)> Voting.deployed().then(function(instance) {console.log(instance.voteForCandidate('Jose', 25).then(function(v) {console.log(v)}))})
truffle(development)> Voting.deployed().then(function(instance) {console.log(instance.voteForCandidate('Rama', 10).then(function(v) {console.log(v)}))})
truffle(development)> Voting.deployed().then(function(instance) {console.log(instance.voteForCandidate('Nick', 10).then(function(v) {console.log(v)}))})
```

- 查询你所投账户的投票人信息（除非用了其他账户，否则你的账户默认是 web3.eth.accounts[0]）

```
truffle(development)> Voting.deployed().then(function(instance) {console.log(instance.voterDetails('0x004ee719ff5b8220b14acb2eac69ab9a8221044b').then(function(v) {console.log(v)}))})
```

- 现在候选人Rama有多少投票？

```
truffle(development)> Voting.deployed().then(function(instance) {instance.totalVotesFor.call('Rama').then(function(i) {console.log(i)})})
```

#### 4.7
现在，已经了解新的投票合约可以如约工作。现在开始构建前端逻辑，以便用户能够通过网页浏览器与合约交互。
![Alt text](/img/013/20180406-11.png)

以下可以自行参照git上的代码

- HTML

> 如果仔细审查代码的话，你会发现网页中已经没有硬编码的值了。候选人的名字将通过向部署好的合约查询来进行填充。
网页也会显示公司发行的股票通证总量，以及已售出和剩余的通证量。

- Javascript

> 通过移除候选者姓名等等的硬编码，我们已经大幅改进了 HTML 文件。我们会使用javascript/web3js来填充 HTML页面里的所有值，并实现查询投票人信息的额外功能。
> 
> 如果对 JavaScript 不太熟悉，这些代码可能略显复杂。那么最好先理解populateCandidates() 函数的实现。
> 
> 实现帮助
> 
1. 创建一个 Voting 合约的实例
2. 在页面加载时，初始化并创建 web3 对象。（第一步和第二步与之前的课程一模一样）
3. 创建一个在页面加载时调用的函数，它需要：
4. 使用 Voting 合约对象，向区块链查询来获取所有的候选者姓名并填充表格。
5. 再次查询区块链得到每个候选人所获得的所有投票并填充表格的列。
6. 填充 token 信息，比如所有初始化的 token，剩余 token，已售出的 token 以及 token 成本。
7. 实现 buyTokens 函数，它在上一节的 html 里面调用。你已经在控制台交互一节中购买了 token。buyTokens 代码与那一节一样不可或缺。
8. 类似地，实现 lookupVoterInfo 函数来打印一个投票人的细节。

和之前一样，执行以下命令进行构建：

```
~/repo/tkapp$ webpack
```

然后进入build目录，启动轻量web服务器：

```
~/repo/tkapp/build$ python -m SimpleHTTPServer
```

如果一切顺利，你可以看到网页，可以输入一个账户地址（投票人的地址），观察他们的投票行为和股票通证数量的变化。 并且可以购买更多的股票通证，为任意候选者投票并查看投票人信息。

现在合约的实现方式，用户购买股票通证并用通证投票。但是他们投票的方式是向合约发送通证。如果他们必须在未来的 选举中投票怎么办？他们所有的通证都转移到了合约中！

进一步改进合约的方式是，添加加入一个方法以便于用户能够在投票结束后拿回他们的通证。

### 五.总结
到这里基本的教程介绍已经结束，虽说是入门级的DAPP开发，对于之前没有接触过的人来说，也是需要花点时间理顺和让代码跑起来的。投票系统这个案例可能会在很多入门教学中都会使用，所以如果需要跟人展示和交流的话，可能换一种场景会比较看上去高大上一点。如同开发手机APP一样，技术总是工具，真正产生价值的，更多是融入技术的思想。

