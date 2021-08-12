# Truffle通过HDWalletProvider实现交易的本地签名

## Truffle简介

Truffle是以太坊上的智能合约的开发工具，类似Java开发中的Maven或者Gradle工具，但是内涵更加丰富，除了编译链接，它还兼顾了智能合约的部署，合约的部署需要适配多样的目标环境，例如本地的Ganache开发网络、以太坊测试网络（Ropsten Rinkeby or Kovan Net）、以太坊主网（Main Net）。Truffle 为此提供了provider配置选项，默认使用Web3JS中定义的`Web3.providers.HttpProvider`，它会使用`host`和`option`选项构造出目标地址`http://<host>:<port>`，如下所示：

```javascript
module.exports = {
  networks: {
    development: {
      host: "127.0.0.1",
      port: 8545,
      network_id: "*" // Match any network id
    }
  }
};
```



## Truffle部署合约的流程



```
//truffle/packages/contract/lib/execute.js

/**
 * Deploys an instance
 * @param  {Object} constructorABI  Constructor ABI segment w/ inputs & outputs keys
 * @return {PromiEvent}             Resolves a TruffleContract instance
 */
deploy: function (constructorABI) {
  const constructor = this;
  const web3 = constructor.web3;

  return function () {
    let deferred;
    const promiEvent = new PromiEvent(false, constructor.debugger, true);

    execute
      .prepareCall(constructor, constructorABI, arguments)
      .then(async ({ args, params, network }) => {
        const { blockLimit } = network;

        utils.checkLibraries.apply(constructor);

        // Promievent and flag that allows instance to resolve (rather than just receipt)
        const context = {
          contract: constructor,
          promiEvent,
          onlyEmitReceipt: true
        };

        const options = {
          data: constructor.binary,
          arguments: args
        };

        const contract = new web3.eth.Contract(constructor.abi);
        params.data = contract.deploy(options).encodeABI();

        params.gas = await execute.getGasEstimate.call(
          constructor,
          params,
          blockLimit
        );

        context.params = params;

        promiEvent.eventEmitter.emit("execute:deploy:method", {
          args,
          abi: constructorABI,
          contract: constructor
        });

        deferred = execute.sendTransaction(web3, params, promiEvent, context); //the crazy things we do for stacktracing...

        try {
          const receipt = await deferred;
          if (receipt.status !== undefined && !receipt.status) {
            const reason = await Reason.get(params, web3);

            const error = new StatusError(
              params,
              context.transactionHash,
              receipt,
              reason
            );

            return context.promiEvent.reject(error);
          }

          const web3Instance = new web3.eth.Contract(
            constructor.abi,
            receipt.contractAddress
          );
          web3Instance.transactionHash = context.transactionHash;

          context.promiEvent.resolve(new constructor(web3Instance));
        } catch (web3Error) {
          // Manage web3's 50 blocks' timeout error.
          // Web3's own subscriptions go dead here.
          await override.start.call(constructor, context, web3Error);
        }
      })
      .catch(promiEvent.reject);

    return promiEvent.eventEmitter;
  };
},
```

当执行truffle deploy时，整体的调用逻辑链是：

deploy() ->  new web3.eth.Contract(constructor.abi)  -> this.sendTransaction() ->  web3.eth.sendTransaction(params) 

-> method.requestManager.send(payload, sendTxCallback) -> Provider.send

## HttpProvider

```
//web3.js/packages/web3-core-method/src/index.js

Method.prototype.buildCall = function () {

		...

        // SENDS the SIGNED SIGNATURE
        var sendSignedTx = function (sign) {

            var signedPayload = { ... payload, 
                method: 'eth_sendRawTransaction',
                params: [sign.rawTransaction]
            };

            method.requestManager.send(signedPayload, sendTxCallback);
        };


        var sendRequest = function (payload, method) {

            if (method && method.accounts && method.accounts.wallet && method.accounts.wallet.length) {
                var wallet;

                // ETH_SENDTRANSACTION
                if (payload.method === 'eth_sendTransaction') {
                    var tx = payload.params[0];
                    wallet = getWallet((!!tx && typeof tx === 'object') ? tx.from : null, method.accounts);


                    // If wallet was found, sign tx, and send using sendRawTransaction
                    if (wallet && wallet.privateKey) {
                        var tx = JSON.parse(JSON.stringify(tx));
                        delete tx.from;

                        if (method.defaultChain && !tx.chain) {
                            tx.chain = method.defaultChain;
                        }

                        if (method.defaultHardfork && !tx.hardfork) {
                            tx.hardfork = method.defaultHardfork;
                        }

                        if (method.defaultCommon && !tx.common) {
                            tx.common = method.defaultCommon;
                        }

                        method.accounts.signTransaction(tx, wallet.privateKey)
                            .then(sendSignedTx)
                            .catch(function (err) {
                                if (typeof defer.eventEmitter.listeners === 'function' && defer.eventEmitter.listeners('error').length) {
                                    try {
                                        defer.eventEmitter.emit('error', err);
                                    } catch (err) {
                                        // Ignore userland error prevent it to bubble up within web3.
                                    }
                                    defer.eventEmitter.removeAllListeners();
                                    defer.eventEmitter.catch(function () {
                                    });
                                }
                                defer.reject(err);
                            });
                        return;
                    }

                    // ETH_SIGN
                } else if (payload.method === 'eth_sign') {
                    var data = payload.params[1];
                    wallet = getWallet(payload.params[0], method.accounts);

                    // If wallet was found, sign tx, and send using sendRawTransaction
                    if (wallet && wallet.privateKey) {
                        var sign = method.accounts.sign(data, wallet.privateKey);

                        if (payload.callback) {
                            payload.callback(null, sign.signature);
                        }

                        defer.resolve(sign.signature);
                        return;
                    }


                }
            }


            return method.requestManager.send(payload, sendTxCallback);
        };
        
        
    ...
    
}
```

```
//web3.js/packages/web3-providers-http/src/index.js

/**
 * Should be used to make async request
 *
 * @method send
 * @param {Object} payload
 * @param {Function} callback triggered on end with (err, result)
 */
HttpProvider.prototype.send = function (payload, callback) {
    var _this = this;
    var request = this._prepareRequest();
    
    ...
    
}
```

> RequestManager是对Provider的封装:

```
//web3.js/packages/web3-core-requestmanager/src/index.js

RequestManager.providers = {
    WebsocketProvider: require('web3-providers-ws'),
    HttpProvider: require('web3-providers-http'),
    IpcProvider: require('web3-providers-ipc')
};
```

## 

根据上面的代码逻辑可以知道，web3.js对于eth_sendTransaction方法的的处理逻辑：

(1)如果可以找到钱包，则使用from参数对应的私钥进行签名，然后调用eth_sendRawTransaction这个RPC接口。

(2)如果没找到钱包，则不做任何处理，直接发送RPC接口。



## HDWalletProvider

HDWalletProvider是Truffle提供的一个组件，web3.js本身是没有的。HDWalletProvider是基于HD Wallet(可以参看BIP32)的Web3 Provider，Wallet即存储私钥，所以不难想象它就是用来对合约交易进行签名的。那么Truffle是如何做的，需要先理解几个概念:

- Web3 provider engine

- HookedSubprovider

  

### Web3 provider engine

要理解Web3 provider engine，要先理解web3和provider，web3是一组和以太坊节点交互的RPC API的定义。provider是就是执行RPC接口请求的程序，比如HttpProvider、WebsocketProvider、IpcProvider，这些都是Web3.js提供的。

Web3 provider engine是用于组合不同Web3 provider的组件，这些provider各自实现了Web3.定义的部分功能，Web3 provider engine是所有provider的入口。

```
//truffle/packages/hdwallet-provider/src/index.ts

import ProviderEngine from "@trufflesuite/web3-provider-engine";
import FiltersSubprovider from "@trufflesuite/web3-provider-engine/subproviders/filters";
import NonceSubProvider from "@trufflesuite/web3-provider-engine/subproviders/nonce-tracker";
import HookedSubprovider from "@trufflesuite/web3-provider-engine/subproviders/hooked-wallet";
import ProviderSubprovider from "@trufflesuite/web3-provider-engine/subproviders/provider";
// @ts-ignore
import RpcProvider from "@trufflesuite/web3-provider-engine/subproviders/rpc";
// @ts-ignore
import WebsocketProvider from "@trufflesuite/web3-provider-engine/subproviders/websocket";
```

### HookedSubprovider

ProviderEngine添加了HookedSubprovider：

```
//truffle/packages/hdwallet-provider/src/index.ts

this.engine.addProvider(
  new HookedSubprovider({
    getAccounts(cb: any) {
      cb(null, tmpAccounts);
    },
    getPrivateKey(address: string, cb: any) {
      if (!tmpWallets[address]) {
        return cb("Account not found");
      } else {
        cb(null, tmpWallets[address].getPrivateKey().toString("hex"));
      }
    },
    async signTransaction(txParams: any, cb: any) {
      await self.initialized;
      // we need to rename the 'gas' field
      txParams.gasLimit = txParams.gas;
      delete txParams.gas;

      let pkey;
      const from = txParams.from.toLowerCase();
      if (tmpWallets[from]) {
        pkey = tmpWallets[from].getPrivateKey();
      } else {
        cb("Account not found");
      }
      const chain = self.chainId;

		...
		
      const tx = new Transaction(txParams, txOptions);
      tx.sign(pkey as Buffer);
      const rawTx = `0x${tx.serialize().toString("hex")}`;
      cb(null, rawTx);
    },
	
	...
	
  })
);
```

在HookedSubprovider的实现中，可以看到`getPrivateKey`和`signTransaction`函数，这两个函数就是为`eth_sendRawTransaction`发送签名后的交易服务的。`signTransaction`函数中使用了ethereumjs-tx库对交易数据进行签名处理。

## 配置 truffle-config.js 配置文件

    const HDWalletProvider = require("truffle-hdwallet-provider");
    
    //privateKey，without 0x prefix
    const privateKeys = '99b3c12287537e38c90a9219d4cb074a89a16e9cdb20bf85728ebd97c343e342'; //address: 0x6Be02d1d3665660d22FF9624b7BE0551ee1Ac91b
    
    module.exports = {
    
      networks: {
        development: {
          network_id: "*",
          provider: () => new HDWalletProvider(privateKeys, `http://127.0.0.1:8545`),
        }
      }


​    
​    };
​    
    ...

## 总结

使用Truffle HDWallet Provider带给开发者极大的方便，使得开发者既不用搭建以太坊节点，也保证私钥的安全，在本地即可对交易进行签名。

