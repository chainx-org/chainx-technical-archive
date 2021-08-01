# The research of frontier

## 1. Introduction
The influence of Ethereum is increasing. As an emerging blockchain development framework, Substrate has to consider how to be compatible with EVM Dapp. Although Parity provides a Bridge method to connect the two blockchain worlds of Ethereum and Polkadot, currently It is only the cross-chain transfer of assets or the cross-chain transfer of messages. In order to support the low-cost migration of Ethereum Dapp to Substrate, Parity launched the [frontier](https://github.com/paritytech/frontier) project at the end of 2018 .

## 2. Ethereum compatibility layer
Frontier builds an Ethereum compatibility layer on substrate, and its goals are:
- Run ordinary Ethereum Dapp through the compatibility layer.
- Can import the status of the Ethereum mainnet


There are three main types of Ethereum Dapp:
- Dapps that do not handle private keys: They rely on eth_sendTransaction and expect RPC nodes to handle private keys.
- Dapps with private keys: These dapps process private keys internally, sign transactions offline and send transactions using eth_sendRawTransaction.
- Specialized dapps: dapps with special functions. These include those Dapps that rely heavily on the internal implementation details of Ethereum. An example is the decentralized mining pool contract.

The goal of the Ethereum compatibility layer is to support dapps with or without private keys. It ignore the case for specialized dapps as it is expensive to emulate subtle details of Ethereum implementations. 

The frontier implementation is mainly composed of the following components:

- runtime: 
  - evm: The reference is [SputnikVM](https://github.com/rust-blockchain/evm) EVM engine
  - pallet-evm: Encapsulates the context in the substrate environment, such as the handling of account balances
  - pallet-ethereum: Store eth transactions and their receipts, convert eth transactions into substrate transactions
- rpc module: ethApi, ethFilterApi, netApi, ethPubSubApi, web3Api

![image](https://user-images.githubusercontent.com/8869892/127769409-30f02364-3db7-488b-b91c-c171d4c73b5a.png)

The processing process of an ordinary transfer transaction:
- ethereum Dapp transactions are sent to pallet-ethereum by eth_sendTransaction or eth_sendRawTransaction rpc call
- pallet-ethereum converts the transaction into a substrate unsigned transaction and calls pallet-evm
- pallet-evm checks the transactions (such as account balance) and executes evm
- evm calls pallet-balances to complete the transfer.

## 3. Current progress
Although the frontier project started earlier, it is still in the POC stage.

Currently in the polkadot community, there are several projects that support EVM:
- Moonbeam:
  Moonbeam promotes the migration of etheteum dapp to substrate completely on the basis of frontier. Its account system adopts the same ecdsa signature algorithm as ethereum at the beginning of its design, including balances, and the decimal is also the same as eth. Its official documents are relatively complete and the development is active.

- Plasm:
  Plasm is also based on the EVM support made by frontier, and there is basically no deletion or modification of frontier.

- Acala:
  Skip frontier's heavy ethereum support, directly based on SputnikVM's own support for evm. Its goal is only to support the deployment and invocation of evm contracts, not eth-compatible APIs.

## 4. the goals and difficulties of Minix evm 
Goals:
- By porting frontier's evm solution, running ordinary ethereum dapp.

Difficulty:
- (1) Debugging of evm contract execution
- (2) Collection of handling fee (conversion of minix 8-digit decimal and eth 18-digit decimal)
