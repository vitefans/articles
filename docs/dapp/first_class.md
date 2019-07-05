# vite dapp 开发入门指南

最近 vite 钱包内的 dapp 越来越多了，社区也越来越关注。

本人开发过一个小游戏，所以写下本文作为一篇入门教程，抛砖引玉，希望能帮到大家。

首先强烈建议大家阅读[官方教程](https://vite.wiki/zh/tutorial/contract/contract.html)、[solidity](https://solidity.readthedocs.io/en/v0.5.10/) 和 [solidity++](https://vite.wiki/zh/tutorial/contract/soliditypp.html) 的语法。

看完官方教程，我们需要知道的是：

1. 什么是异步  
    假如用户 A 给 B 发送了一笔交易：  
    那么 A 的账户链上会产生一个 sendBlock，记录了交易发送方 A 的地址、接收方 B 的地址、转账的 tokenId、转账金额、A 的签名等信息。  
    用户 B 的账户链则会产生一个 receiveBlock，它的 `fromBlockHash` 字段记录了 sendBlock 的 hash，如果是合约调用，还会有一些合约执行结果相关的字段。  
    sendBlock 的 `receiveBlockHash` 则是 receiveBlock 的 hash。

2. 合约的成本及配额  
    线上部署合约需要消耗 10 vite，部署合约需要配额，约 85 配额 1 byte。  
    合约接收交易需要抵押配额

3. 使用 [vscode-soliditypp](https://marketplace.visualstudio.com/items?itemName=ViteLabs.soliditypp) 调试合约

4. 使用[测试钱包](https://vite.wiki/zh/tutorial/contract/testdapp.html)（仅支持 ios）、[@vite/bridge](https://www.npmjs.com/package/@vite/bridge) 获取账户信息


### 合约

下面我们就以一个简单的猜骰子点数的小游戏作为例子，讲解开发步骤。
游戏逻辑如下：
1. 用户调用合约的 `roll` 方法并传递猜的点数，如果猜中给用户双倍奖励。
2. 合约的 offchain method `getState` 返回猜中次数。

合约代码如下：
```solpp
// 该文件路径 ～/Desktop/guess.solpp

pragma soliditypp ^0.4.2;

contract Guess {
    tokenId _vToken = "tti_5649544520544f4b454e6e40"; // 参与代币
    uint256 winCount;

    // 声明 2 个事件
    // indexed 表示该参数可以被索引，在 topics 参数中作为查询条件
    event _roll(address indexed addr, uint guessNo, uint256 amount, bytes32 requestHash);
    event _win(address indexed addr, uint luckyNo, uint256 winAmount);

    onMessage roll(uint guessNo) payable {  // roll 方法可以被用户调用，接收一个参数，即用户猜的点数
        require(msg.amount > 0);            // require 类似我们写测试代码的 assert，条件不满足时，停止执行
        require(msg.tokenid == _vToken);

        emit _roll(msg.sender, guessNo, msg.amount, fromhash());    // 产生 _roll 事件

        uint64 random = random64();         // 获取随机数，随机数实现文档 https://vite.wiki/zh/vep/vep-12.html
        uint No = 1 + random % 6;

        if (No == guessNo) {                // 如果用户猜中了
            uint256 winAmount = msg.amount * 2;

            uint256 rewardPool = balance(_vToken);  // 获取合约账户的余额

            if (rewardPool < winAmount) {
                winAmount = rewardPool;
            }

            msg.sender.transfer(_vToken, winAmount);    // 给用户转账
            emit _win(msg.sender, No, winAmount);       // 产生 _win 事件

            winCount += 1;
        }
    }

    getter getState() returns (uint256) {   // getter 方法，返回猜中的次数
        return (winCount);
    }
}
```

### vscode-soliditypp

vite 官方提供了 vs code 插件 [vscode-soliditypp](https://marketplace.visualstudio.com/items?itemName=ViteLabs.soliditypp)，[参考教程](https://vite.wiki/zh/tutorial/contract/debug.html#vs-code%E6%8F%92%E4%BB%B6)使用插件来部署并调试合约是十分方便的。

该插件的原理是在本地启动一个 gvite 节点，http-rpc 端口是 23456， ws 端口是 23457，然后会创建一个账户，并通过创世账户给这个账户转 100万 vite，我们可以用这个账户来部署、调用合约。

使用手机测试钱包调试时，将手机和电脑连接到同一个 wifi，然后将测试钱包的 RPC URL 设置为 `http://${电脑的IP}:23456`，即可连接到本地节点。

在电脑上可以使用 postman 访问 http://127.0.0.1:23456 调用 rpc 接口，获取该本地节点的一些调试信息。

另外，因为插件内包含了合约部署、调用、结果解析等逻辑，所以强烈建议前端开发者参考[源码](https://github.com/vitelabs/soliditypp-vscode)，主要是 view 目录。


### 编译合约

安装 vscode-soliditypp 之后，插件会自动下载 solppc 编译器到 `~/.vscode/extensions/vitelabs.soliditypp-${version}/bin/solppc` 目录。

我当前安装的插件版本是 0.4.3，所以进入到 ~/.vscode/extensions/vitelabs.soliditypp-0.4.3/bin/solppc 目录，执行 `./solppc --bin --abi ~/Desktop/guess.solpp` 编译合约代码，得到结果如下：

```
Binary: 
6080604052695649544520544f4b454e6000806101000a81548169ffffffffffffffffffff021916908369ffffffffffffffffffff16021790555034801561004657600080fd5b5061026f806100566000396000f3fe608060405260043610610041576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff168063df7ddfa914610046575b600080fd5b6100726004803603602081101561005c57600080fd5b8101908080359060200190929190505050610074565b005b60003411151561008357600080fd5b6000809054906101000a900469ffffffffffffffffffff1669ffffffffffffffffffff164669ffffffffffffffffffff161415156100c057600080fd5b3374ffffffffffffffffffffffffffffffffffffffffff167fd0283d410d96b7bbaf99b72147cbe86dd269340ab0cdda3497f8abd686d0d9c182344960405180848152602001838152602001828152602001935050505060405180910390a260004a9050600060068267ffffffffffffffff1681151561013c57fe5b0660010167ffffffffffffffff1690508281141561023e57600060023402905060008060009054906101000a900469ffffffffffffffffffff1631905081811015610185578091505b3374ffffffffffffffffffffffffffffffffffffffffff166000809054906101000a900469ffffffffffffffffffff1669ffffffffffffffffffff168360405160405180820390838587f1505050503374ffffffffffffffffffffffffffffffffffffffffff167fde5035b37239af801ef034afd8cb4d35c43e570ac3af05461baddcaed04b27b48484604051808381526020018281526020019250505060405180910390a26001806000828254019250508190555050505b50505056fea165627a7a72305820f90d31ebd258c2a9544f2539553473e06e53a605b97f4c5260061cfcd89776500029
OffChain Binary: 
6080604052600436106042576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff1680636c49a7be146044576042565b005b604a6060565b6040518082815260200191505060405180910390f35b60006001600050549050606e565b9056fea165627a7a72305820f90d31ebd258c2a9544f2539553473e06e53a605b97f4c5260061cfcd89776500029
Contract JSON ABI 
[{"constant":true,"inputs":[],"name":"getState","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"offchain"},{"constant":false,"inputs":[{"name":"guessNo","type":"uint256"}],"name":"roll","outputs":[],"payable":true,"stateMutability":"payable","type":"function"},{"anonymous":false,"inputs":[{"indexed":true,"name":"addr","type":"address"},{"indexed":false,"name":"guessNo","type":"uint256"},{"indexed":false,"name":"amount","type":"uint256"},{"indexed":false,"name":"requestHash","type":"bytes32"}],"name":"_roll","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"addr","type":"address"},{"indexed":false,"name":"luckyNo","type":"uint256"},{"indexed":false,"name":"winAmount","type":"uint256"}],"name":"_win","type":"event"}]
```

Binary 是合约二进制代码的十六进制格式，长度为 1418，即 709 byte，那么部署需要的配额约为 60265，参考[配额文档](https://vite.wiki/zh/tutorial/rule/quota.html)，至少需要为合约部署账户抵押 400 vite。

OffChain Binary 是所有 getter 方法的编译结果，本例中只有一个 getter，因此 OffChain Binary 也就是 getState 方法。

Contract JSON ABI 是合约方法、事件的 api 描述，调用和结果解析时需要使用。

### 部署合约

如果是使用插件调试，可以先跳过部署阶段，但是线上环境还是需要自己写部署代码的。

```js
const { HTTP_RPC } = require('@vite/vitejs-http');
const { constant, utils, client, hdAccount, abi } = require('@vite/vitejs');

// 使用测试钱包创建一个账户作为合约的创建者，将助记词粘贴到这里
const OWNER_MEM = 'xxxxx';

// 将插件启动的测试节点作为 rpc 服务，线上环境需要替换成自己的 api 节点
// node 运行没有跨域问题
// 如果是前端页面就需要代理了，最简单的就是用 nginx
const httpProvider = new HTTP_RPC('http://127.0.0.1:23456');

const vclient = new client(httpProvider, () => {
    console.log("client connected");
});

// 通过助记词创建账户，文档 https://vite.wiki/zh/api/vitejs/wallet/hdAccount.html
const myHdAccount = new hdAccount({
    mnemonic: OWNER_MEM,
    client: vclient
});

// 由助记词派生出地址
const myAccount = myHdAccount.getAccount({
    index: 0
});

const CONTRACT = {
    binary: '6080 ... 0029',    // 编译的 Binary
    abi: [],                    // 编译的 Contract JSON ABI
    offChain: '6080 ... 0029',  // 编译的 OffChain Binary
    address: '',                // 部署之后的地址
}

async function createContract (binary, abi, params) {
    // 创建合约的参数参考：https://vite.wiki/zh/api/rpc/contract.html#contract-getcreatecontractdata

    // vitejs@2.1.2 版本
    // times 是配额翻倍数
    // confirmTimes 是 sendBlock 被确认的次数
    const optionsV212 = {
        times: 10,
        confirmTimes: 12,
        hexCode: binary,
        abi: abi,
        params: params,
    }

    // vitejs@2.2.2 版本
    // vitejs 新版本参数上有变更：
    // times -> quotaRatio
    // confirmTimes -> confirmTime
    // 新增 seedCount 参数，需要小于或等于 confirmTime
    const optionsV222 = {
        quotaRatio: 10,
        confirmTime: 12,
        seedCount: 12,
        hexCode: binary,
        abi: abi,
        params: params,
    }

    const createTx = await myAccount.createContract(optionsV222);

    const createBlock = await vclient.request('ledger_getBlockByHeight', createTx.accountAddress, createTx.height);

    // 合约地址
    CONTRACT.address = createBlock.toAddress;

    return createBlock
}

// 部署合约
createContract(CONTRACT.binary, CONTRACT.abi, [])
.then(res => console.log(res))
.catch(err => console.error(err))
```

在本地的测试环境，我们需要使用创世账户给自己转账，否则没有余额来部署和测试，转账代码如下：
```js
const { account } = require('@vite/vitejs');

// 本地测试节点的创世块私钥
const GENESIS_PRIVATEKEY = '7488b076b27aec48692230c88cbe904411007b71981057ea47d757c1e7f7ef24f4da4390a6e2618bec08053a86a6baf98830430cbefc078d978cf396e1c43e3a';

const genesisAccount = new account({
    privateKey: GENESIS_PRIVATEKEY,
    client: vclient,
})

// 接收转账
async function receiveAllOnroadTx(account) {
    const blocks = await vclient.onroad.getOnroadBlocksByAddress(account.address, 0, 10);

    for (let i = 0 ; i < blocks.length; i++) {
        const block = blocks[i];
        // receive on road
        await account.receiveTx({
            fromBlockHash: block.hash
        });
    }

    if (blocks.length >= 10) {
        return await receiveAllOnroadTx(account);
    }
}

function viteToString(amount) {
    return amount + '000000000000000000'
}

receiveAllOnroadTx(genesisAccount)  // 先接收创世账户的转账，保证创世账户有钱
.then(() => genesisAccount.sendTx({ // 再给合约创建账户转账
    toAddress: myAccount.address,
    tokenId: constant.Vite_TokenId,
    amount: viteToString(10000),  // 10000 vite 必须是字符串
}))
.then(() => receiveAllOnroadTx(myAccount))  // 再接收合约创建账户
.catch(err => console.error(err))
```

转账之后，这样我们的测试账户就有钱部署合约并测试了。

转账功能已经咨询过插件开发者，预计会在插件的下个版本加上。

线上部署需要注意几点：
1. 需要连接到自己到 api 节点
2. 合约部署账户需要有至少 10 vite 作为创建费用，并抵押配额来部署合约

### offchain method

offchain method 不会被部署到链上，调用比较简单，原理是将代码发送给 api 节点执行，返回执行结果。

由于每次调用需要将 offchain 二进制代码发送到 api 节点，所以大家要注意带宽的使用。

目前合约内的多个 getter 代码会被编译到一起，假如合约内有 3 个 getter A B C，那么调用 A 也会把 B 和 C 的代码发给 api，会损耗带宽，增加请求的时长。

因为 getter 的增删并不会影响 Binary 和 abi，所以这种情况有一个解决方案：将源码中的 B 和 C 注释掉，重新编译源码，得出的 offchain binary 就只是 A 方法的了，同理可以拿到 B 和 C 的二进制代码。

这样调用 A 就只发 A 的代码，会明显减少带宽的损耗。

下面是 offchain method 调用代码：
```js
function getMethodABI(methodName) {
    for (let i = 0; i < CONTRACT.abi.length; i++) {
        const abi = CONTRACT.abi[i]
        if (abi.name === methodName) {
            return abi
        }
    }

    throw new Error('no such method: ' + methodName)
}

async function callContractOffChainMethod(methodName, methodParams) {
    // 获取调用方法的 abi
    const methodABI = getMethodABI(methodName)

    // abi 包由 vitejs 导入
    // 根据 abi 序列化参数，得到十六进制字符串
    const data = abi.encodeFunctionCall(methodABI, methodParams);

    // utils 包由 vitejs 导入
    // 将十六进制转换为 base64 格式
    const dataBase64 = utils._Buffer.from(data, 'hex').toString('base64');

    const result = await vclient.request('contract_callOffChainMethod', {
        'selfAddr': CONTRACT.address,
        'offChainCode': CONTRACT.offChain,
        'data': dataBase64
    });

    // 解析结果
    if (result) {
        // 得到的 result 是 base64 格式，转换为十六进制
        const bytes = utils._Buffer.from(result, 'base64').toString('hex');
        const outputs = [];
        for (let i = 0; i < methodABI.outputs.length; i++) {
            outputs.push(methodABI.outputs[i].type);
        }

        // 根据 output 的类型反序列化 result
        const offchainDecodeResult = abi.decodeParameters(outputs, bytes);
        const resultList=[];
        for (let i = 0; i < methodABI.outputs.length; i++) {
            if (methodABI.outputs[i].name) {
                resultList.push({
                    'name': methodABI.outputs[i].name,
                    'value': offchainDecodeResult[i]
                });
            } else {
                resultList.push({
                    'name': '',
                    'value':offchainDecodeResult[i]
                });
            }
        }

        return resultList;
    }

    return null;
}

callContractOffChainMethod('getState', [])
.then(result => console.log(result))
.catch(err => console.error(err))
```

### 调用合约方法

非 offchain method 是通过向合约账户发送交易来调用的。

合约部署之后，需要检查合约账户链的高度，至少为 1 才表示合约创建成功，才能调用合约方法。

下面是调用代码：
```js
// 获取合约最新块
function getContractLatestBlock() {
    return vclient.request('ledger_getLatestBlock', CONTRACT.address)
}

// 轮询合约最新块
function waitForContractCreated() {
    return new Promise(resolve => {
        function check() {
            getContractLatestBlock().then(block => {
                if (block && block.height > 0) {
                    resolve(block)
                } else {
                    setTimeout(check, 200)
                }
            })
        }

        check()
    })
}

// 调用合约，并返回拼装的 sendBlock
async function callContractMethod(methodName, methodParams, amount) {
    const methodABI = getMethodABI(methodName);

    // 发送交易
    const sendTx = await myAccount.callContract({
        tokenId: constant.Vite_TokenId, // constant 由 vitejs 导入
        amount: amount,     // amount 是转账金额，即合约代码中的 msg.amount
        abi: methodABI,
        params: methodParams,
        toAddress: CONTRACT.address,
    })

    return sendTx
}

// 根据拼装的 sendBlock 查询 receiveBlock
function checkReceive(accountAddress, height) {
    return new Promise((resolve, reject) => {
        async function check() {
            // 查询 sendBlock 是否处理
            const sendBlock = await vclient.request('ledger_getBlockByHeight', accountAddress, height);

            if (!sendBlock) {
                setTimeout(check, 1000)
                return
            }

            // 查询合约是否接收，并产生 receiveBlock
            const receiveBlockHash = sendBlock.receiveBlockHash;
            if (!receiveBlockHash) {
                setTimeout(check, 1000)
                return
            }

            const receiveBlock = await vclient.request('ledger_getBlockByHash', receiveBlockHash);
            if (!receiveBlock) {
                setTimeout(check, 1000)
                return
            }

            resolve(receiveBlock)
        }

        setTimeout(check, 1000)
    })
}

waitForContractCreated()    // 等待合约创建完成
.then(() => callContractMethod('roll', [1], viteToString(10)))  // 调用合约 roll 方法，猜点数为 1，支付 10 vite
.then(sendTx => checkReceive(sendTx.accountAddress, sendTx.height)) // 轮询 receiveBlock
.then(logs => console.log(logs))
.catch(err => console.error(err))
```

如果在 wallet app 内通过 bridge 发送交易的话，代码如下：
```js
import Bridge from '@vite/bridge';

let account_address = ''
const bridge = new Bridge()
const getAccountPro = bridge['wallet.currentAddress']().then(res => {
    account_address = res   // 获取当前账户地址
})

// uri 拼接文档：https://vite.wiki/zh/vep/vep-6.html
async function callContractMethod(methodName, methodParams, amount) {
    const methodABI = getMethodABI(methodName);

    const hexFunctionCall = abi.encodeFunctionCall(methodABI, methodParams);
    const base64FunctionCall = utils._Buffer.from(hexFunctionCall, 'hex').toString('base64');

    // 通过 bridge 发送交易
    const sendTx = await bridge["wallet.sendTxByURI"]({
        address: account_address,
        // 拼接 uri
        uri: utils.uriStringify({
            target_address: CONTRACT.address,
            function_name: methodName,
            params: {
                tti: constant.Vite_TokenId,
                amount: amount, // 注意，这里的 amount 单位是 vite
                data: base64FunctionCall,
            }
        })
    })

    return sendTx
}

getAccountPro   // 获取钱包当前地址
.then(() => waitForContractCreated())   // 等待合约创建完成
.then(() => callContractMethod('roll', [1], 10))  // 调用合约 roll 方法，猜点数为 1，支付 10 vite
.then(sendTx => checkReceive(sendTx.accountAddress, sendTx.height)) // 轮询 receiveBlock
.then(logs => console.log(logs))
.catch(err => console.error(err))
```

### 处理 log

合约的 roll 方法会产生 _roll 事件，并在用户猜中时，会产生 _win 事件，这些事件会作为 log 被节点存储下来。

良好的体验是如果用户中奖了，前端则提示用户中奖，那么就需要根据 receiveBlock 的 hash 获取并解析事件内容。

log 相关文档 https://vite.wiki/zh/api/rpc/subscribe.html

```js
async function fetchLog(receiveHash) {
    // 根据 receiveBlock 的 hash 查询 log
    const vmLogList = await vclient.request('ledger_getVmLogList', receiveHash);
    return parseVMLogs(vmLogList)
}

// 根据事件 abi 获取事件签名
const events = getLogEventsSignature()
function getLogEventsSignature () {
    const events = {}

    for (let i = 0; i < CONTRACT.abi.length; i++) {
        const item = CONTRACT.abi[i]
        if (item.type === 'event') {
            events[item.name] = Object.assign({}, item, {
                signature: abi.encodeLogSignature({
                    'type': 'event',
                    'name': item.name,
                    'inputs': item.inputs,
                })
            })
        }
    }

    return events
}

function parseVMLogs(vmLogList) {
    const vmLogs = [];

    if (vmLogList && vmLogList.length > 0) {
        vmLogList.forEach(vmLog => {
            vmLogs.push(parseVMLog(vmLog))
        });
    }

    return vmLogs
}

/**
 * vmLog 格式如下：
 * {
        "topics": [
            "6639390f62a7d839beee30a534322ec7723aefb701fb097e4a5c63aefe4b03f2"
        ],
        "data": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUhKua77xuLtXDdw1s4rqtVT+X8wmMAq6mccFUBZ+BRQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYuEVwAiogAA"
    }

 * topics 文档 https://vite.wiki/zh/api/rpc/subscribe.html#subscribe-newlogsfilter
 * topics 是一个二维数组
 * [
 *      [],     // 第一个数组，元素是事件签名
 *      [],     // 第二个数组，元素 indexed 的参数
 * ]
 */
function parseVMLog(vmLog) {
    const topics = vmLog.topics;
    for (let name in events) {
        if (events[name].signature === topics[0]) {
            let data = '';
            if (vmLog.data) {
                data = utils._Buffer.from(vmLog.data, 'base64');
            }

            return {
                topic: topics[0],
                args: abi.decodeLog(events[name].inputs, data.toString('hex'), topics.slice(1)),
                event: name
            }
        }
    }
}

// 调用并解析 log 的完整过程
callContractMethod('roll', [1], 10 * 10 ** 18)  // 调用合约 roll 方法，猜点数为 1，支付 10 vite
.then(sendTx => checkReceive(sendTx.accountAddress, sendTx.height)) // 轮询 receiveBlock
.then(receiveBlock => fetchLog(receiveBlock.hash))
.then(logs => {  // 解析的 logs 是 roll 产生的所有事件，包括 _roll 和 _win
    let win = false
    for (let i = 0; i < logs.length; i++) {
        if (logs[i].event === '_win') { // 如果有 _win 事件，表示中奖了
            win = true
            console.log('恭喜您中奖了，奖金', logs[i].args.winAmount)
            break
        }
    }

    if (!win) {
        console.log('很遗憾，您没中奖')
    }
})
.catch(err => console.error(err))
```

### 获取记录

因为一个 receiveBlock 可能会有一个中奖记录，那么需要获取这个合约的所有中奖记录，就需要遍历 receiveBlock 了。

目前 gvite 的查询有个[小 bug](https://github.com/vitelabs/go-vite/pull/376)，建议高度分批次 100 查询。

当游戏参与人数很多的时候，合约链会增长非常快，遍历查询的性能就会比较差。建议前端在 localStorage 缓存上次查询的高度，然后从最新高度向后查询，直到上次查询的高度（考虑到链可能会回滚，截止的高度最好是比上次缓存的高度低）。

下面是查询代码：
```js
// from to 是合约链的块高度，取 0 的时候表示从第一个块到最新的块
async function getContractLog(topics, from = 0, to = 0) {
    const result = await vclient.request('subscribe_getLogs', {
        "topics": topics,
        "addrRange": {
            [CONTRACT.address]: {
                "fromHeight": "" + from,
                "toHeight": "" + to,
            }
        }
    })

    if (!result) {
        return []
    }

    const resultLogs = result.map(item => item.log)
    return parseVMLogs(resultLogs)
}

// 获取所有账户的中奖纪录
getContractLog([
    [events._win.signature]
], 0, 99)
.then(res => {
    res.forEach(item => {
        console.log(`中奖地址${item.addr} 点数${item.luckyNo} 金额${item.winAmount}`)
    })
})
.catch(err => console.error(err))

// 获取自己的中奖记录
getContractLog([
    [events._win.signature],
    [abi.encodeParameter('address', myAccount.address)]
], 0, 99)
.then(res => console.log(res))
.catch(err => console.error(err))
```

至此，合约的部署、调用、log 等功能已经都讲完了。

### api

目前 gvite 客户端具备 http-rpc 功能，也就是说你运行的 vite 节点就可以作为 API 节点来使用，所以一个 dapp 大概的系统调用流程如下：

![架构](https://raw.githubusercontent.com/vitefans/articles/master/docs/dapp/1.png)

不过由于合约执行、存储需要的配额很高，所以一般合约开发者只会存核心数据。开发者可以搭建自己的中间 API 对数据进行处理，方便前端的查询。架构就会变成这样：

![架构](https://raw.githubusercontent.com/vitefans/articles/master/docs/dapp/2.png)

开启 gvite 的 API 功能需要在设置文件 `node_config.json` 配置相应字段:

| 字段名称 | 值 | 说明 |
|:--|:--:|:--|
| RPCEnabled | true | 表示开启 |
| HttpPort | 自定义 | rpc 的端口 |
| PublicModules | "ledger", "private_onroad", "net", "contract", "pledge", "register", "vote", "mintage", "consensusGroup", "tx", "debug", "sbpstats", "pow", "subscribe" | 开启哪些模块的 rpc 功能 |
| OpenPlugins | true | 在途资金相关功能 |
| VmLogAll | true | 存储合约 log |

如果需要 websocket 功能，则需要配置以下字段
- WSEnabled: true
- WSPort: 自定义


### 数据

再稍微提一下前端数据处理的问题，gvite 处理的数据很多都是字节数组，但是转换为 json 数据之后，js 读取到的格式会不一样，比如：

| 字段 | 原始类型 | json 类型 | json 举例 |
|:--:|:--:|:--:|:--|
| hash | [32]byte | hex string | 558f9c10177e3bedfa3e06d4d811f31e851f5bc2d52eb8214b28066f476fde10 |
| signature | [64]byte | base64 | //47ziCwKjzcKNZPPwHtnj/bk/XIE9E5rr1VGlMkRaBCiJJQet5W7Y4nWlpWNyRhv0EJBvmGAB5KGPHpkqUcBw== |
| height | uint64 | string | 277 |
| address | [21]byte | address string | vite_bb34549c9c0a7987b7ac4e0e8bab7fddc8ce04f79e3af4cd56 |

为什么字节数组转换会有不同的规则呢？有的是 hex string，有的是 base64 呢？

因为对于一些常见的字段，比如 hash 和 address，gvite 做了特殊处理，方便识别与 debug。而 uint64 转换为字符串是避免 js 精度移除。

剩下没有特殊处理的字节数组，会被自动转换为 base64 格式，比如 signature。

举个例子，某合约的 receiveBlock json 关键结构:
```json
{
    "blockType": 4,
    "height": "277",
    "hash": "558f9c10177e3bedfa3e06d4d811f31e851f5bc2d52eb8214b28066f476fde10",
    "fromAddress": "vite_f935004e521b9989540a26dbe4a79c5b11e35599b17b8ee516",
    "toAddress": "vite_bb34549c9c0a7987b7ac4e0e8bab7fddc8ce04f79e3af4cd56",
    "fromBlockHash": "91cbb9ebe348347ad4fef3966cef9870b12f302a27f21a91a127acb315f42ab0",
    "tokenId": "tti_5649544520544f4b454e6e40",
    "amount": "50000000000000000000",
    "data": "sRe28/sjj+l+pv4BdDXiRJGVe7HrteNMnArbZdr931gA",
    "signature": "//47ziCwKjzcKNZPPwHtnj/bk/XIE9E5rr1VGlMkRaBCiJJQet5W7Y4nWlpWNyRhv0EJBvmGAB5KGPHpkqUcBw==",
    "quota": "60446",
    "quotaUsed": "60446",
    "logHash": "01c5b75ecae7e78dd10ac32f16ee26ecff8dba7450b4cba3f37393327bea1aa3",
    "confirmedTimes": "91",
    "confirmedHash": "d4d81b9ca6b81b8fe4a6521c46e0e294309412a8764f1396e6538b2a1438ad1d",
    "timestamp": 1561687070
}
```

关键字段的解释:
1. fromAddress 是交易发送方的地址
2. toAddress 是交易接收方的地址，也就是合约地址
3. fromBlockHash 交易的 sendBlock
4. tokenId 支付的代币，VITE 是 `tti_5649544520544f4b454e6e40`
5. amount 这笔交易的金额，除以 `10**18` 可以转换为 VITE 单位

解析数据时，可以用
```js
// base64 -> ArrayBuffer
utils._Buffer.from(base64_data, 'base64');

// hex -> ArrayBuffer
utils._Buffer.from(hex_data, 'hex');

// hex -> base64
utils._Buffer.from(hex_data, 'hex').toString('base64');
```

本文涉及到到的代码较多，建议大家尝试运行，如果有问题，请在帖子下留言沟通，谢谢。

补充：
文中所有的 vitejs 版本号是 2.1.2。  
发文不久，在调试另一个 dapp 时，合约总是创建失败。各种调试最后发现安装的是 vitejs@2.2.2 版本，再定睛一看，原来创建合约的参数有变更，但是文档没有提到，故而踩了一坑。

所以更新了文中创建合约的代码。。。
