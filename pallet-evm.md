# pallet-evm: An implementation of EVM in Substrate

## Background

Since the advent of Ethereum, it has been unanimously recognized by the blockchain industry, and smart contracts have also detonated many tracks in recent years. From Defi last year to the popularity of NFT today, Ethereum cannot be avoided. Now Ethereum's market value is firmly in second place, closely behind Bitcoin. And compared to Bitcoin, Ethereum's distributed application (dapp) can bring more imagination to the market. The core of Ethereum is its evm, which can execute smart contracts, and pallet-evm is dedicated to implementing evm on the substrate to introduce Ethereum's smart contracts painlessly.

## Instructure of EVM

In Ethereum, there are two type of transaction, which are contract creation and message call. Contract creation results in a smart contract creation, and message call results in message sending to a smart contract. A smart contract executes as below:

![evm instructure](https://ethereum.org/static/9628ab90bfd02f64cf873446cbdc6c70/302a4/gas.png)

The execution of smart contract has three main components：

- EVM code
- Machine State
- Storage



EVM code is the byte code compiled by solidity, which will load into the ROM when smart contract executing. EVM code initialized by contract creation transaction, and cannot be modified later. 



Storage is data-persistent component, which is a part of Ethereum's World State. It's associated with each account.  Whether more, storages form every account are standalone. 



Machine State is the place actually execute evm code.  It's consist of pc, stack and memory. PC is same as the program counter register in computer science. EVM code divides into many operations, which the machine executes one by one.  

## What the pallet-evm does?

In a word, pallet-evm allows unmodified EVM code to be executed in a Substrate-based blockchain.



According to the EVM yellow paper, the Storage is key-value pair list, while key and value are both 256-bits word, and the Evm code is a bounded byte array. Thus the pallet-evm simulates them by a storage map while key is the account.   As for the EVM engine, pallet-evm use [SputnikVM](https://github.com/rust-blockchain/evm) instead.



In many situations, a Substrate blockchain may only want to include EVM execution capatibilities. In this way, it functions similarly to `pallet-contracts`, integrates with Substrate better and is less intrusive. The module, and its EVM execution capatibilties, can be added or removed at any moment via forkless upgrades. With EVM execution only, Substrate uses its account model fully and signs transactions on behalf of EVM accounts.

In this model, however, Ethereum RPCs are not available, and dapps must rewrite their frontend using the Substrate API.



## pallet-evm vs Ethereum network

The EVM module should be able to produce nearly identical results compared to the Ethereum mainnet, including gas cost and balance changes.

Observable differences include:

- The available length of block hashes may not be 256 depending on the configuration of the System module in the Substrate runtime.
- Difficulty and coinbase, which do not make sense in this module and is currently hard coded to zero.

The module currently do not aim to make unobservable behaviors, such as state root, to be the same. Also don't aim to follow the exact same transaction / receipt format. However, given one Ethereum transaction and one Substrate account's private key, one should be able to convert any Ethereum transaction into a transaction compatible with this module.

The gas configurations are configurable. Right now, a pre-defined Istanbul hard fork configuration option is provided.



## Conclusion

Pallet-evm is a great and successful idea to bring Ethereum into Polkadot ecosystem.  There are many projects benefit from that, such as Acala, Moonbeam and plasm. It brings more flexibility to our blockchain, which coming chat needs. 