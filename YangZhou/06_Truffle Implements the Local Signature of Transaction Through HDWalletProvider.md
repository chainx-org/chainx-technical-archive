# Truffle Implements the Local Signature of Transaction Through HDWalletProvider



Truffle is an Ethereum smart contract integrated development environment, which aims to simplify the work of developers and improve development efficiency.

Truffle has the following features:

- Built-in smart contract compilation, linking, deployment and binary management.
- Automated contract testing for rapid development.
- Scriptable, extensible deployment & migrations framework.
- Network management for deploying to any number of public & private networks, such as local Ganache development network, Ethereum test network and Ethereum main network.
- Interactive console for direct contract communication.
- and so on

## Truffle Deployment Contract Process

Analyze Truffle source code:

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

		...

        const contract = new web3.eth.Contract(constructor.abi);
        params.data = contract.deploy(options).encodeABI();

		...

        deferred = execute.sendTransaction(web3, params, promiEvent, context); //the crazy things we do for stacktracing...

        try {
          const receipt = await deferred;
 
 		  ...
 	
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

...

  sendTransaction: async function (web3, params, promiEvent, context) {
    //if we don't need the debugger, let's not risk any errors on our part,
    //and just have web3 do everything
    if (!promiEvent || !promiEvent.debug) {
      const deferred = web3.eth.sendTransaction(params);
      handlers.setup(deferred, context);
      return deferred;
    }
    //otherwise, do things manually!
    //(and skip the PromiEvent stuff :-/ )
    return sendTransactionManual(web3, params, promiEvent);
  }
```

It can be seen that when `truffle deploy` is executed, the overall calling logic chain is:

deploy() ->  new web3.eth.Contract(constructor.abi)  -> this.sendTransaction() ->  web3.eth.sendTransaction(params) 

-> method.requestManager.send(payload, sendTxCallback) -> Provider.send



## HttpProvider

Let's first look at the HttpProvider used by web3.js default:

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
						 
						 ...

                        method.accounts.signTransaction(tx, wallet.privateKey)
                            .then(sendSignedTx)
                            .catch(function (err) {
								
								...
								
                            });
                        return;
                    }

                    // ETH_SIGN
                } else if (payload.method === 'eth_sign') {
				
					 ...
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

> Requestmanager is the encapsulation of provider:

```
//web3.js/packages/web3-core-requestmanager/src/index.js

RequestManager.providers = {
    WebsocketProvider: require('web3-providers-ws'),
    HttpProvider: require('web3-providers-http'),
    IpcProvider: require('web3-providers-ipc')
};
```

According to the above code logic, processing logic of  eth_sendTransaction method for web3.js:

(1) If  the wallet is found, use the private key corresponding to the from parameter to sign it, then call eth_sendRawTransaction this RPC interface.

(2) If the wallet is not found, it will be sent directly to the RPC interface without any processing.



## HDWalletProvider

HDWalletProvider is a component provided by Truffle, Web3.js itself does not has. HDWalletProvider is a Web3 provider based on HD wallet (see bip32). Wallet stores the private key, so it is not hard to know that it is used to sign contract transactions. So how truffle does? it needs to understand several concepts first:

- Web3 provider engine

- HookedSubprovider

### Web3 provider engine

To understand Web3 provider engine, you must first understand Web3 and provider. Web3 is a set of RPC APIs that interact with Ethereum nodes. Providers are programs that execute RPC interface requests, such as HttpProvider, WebsocketProvider and IpcProvider. These providers are provided by web3.js.

Web3 provider engine is a component used to combine different Web3 providers. These providers implement some of the functions defined by Web3.  Web3 provider engine is the entrance of all providers.

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

...

```

### HookedSubprovider

Truffle adds HookedSubprovider to ProviderEngine:

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

In the implementation of HookedSubprovider, you can see the `getPrivateKey` and `signTransaction` functions, which are support for `eth_sendRawTransaction`. The ethereumjs-tx library is used in the `signTransaction` function to sign transaction data.

## Configure the truffle-config.js 

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
    };

The parameter of HDWalletProvider:

first parameter:  mnemonic or private key

second parameter: URI or Ethereum client to send Web3 requests



## Summary

Using Truffle HDWalletProvider brings great convenience to developers, so that developers do not need to build Ethereum nodes, but also ensure the security of private keys, so that they can sign transactions locally.

