# Introduction
As we all know, Bitcoin is known as the earliest and largest distributed ledger. While Ethereum has its own native cryptocurrency, it also enables a much more powerful function: smart contracts. For this more complex feature, a more sophisticated analogy is required. Instead of a distributed ledger, Ethereum is a distributed state machine. Ethereum's state is a large data structure which holds not only all accounts and balances, but a machine state, which can change from block to block according to a pre-defined set of rules, and which can execute arbitrary machine code. The specific rules of changing state from block to block are defined by the EVM.


# EVM Overall Architecture

Ethereum bottom layer supports contract execution and calling through EVM module. When calling, the contract code is obtained according to the contract address and loaded into EVM for operation. Usually, the development process of smart contract is to write logic code with solidity, compile it into bytecode through compiler, and finally release it to Ethereum.

The EVM is not only sandboxed but actually completely isolated, which means that code running inside the EVM has no access to network, filesystem or other processes. Smart contracts even have limited access to other smart contracts.

![在这里插入图片描述](https://img-blog.csdnimg.cn/59ec48be1ff14530841d6f59e338115e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l6cGJyaWdodA==,size_16,color_FFFFFF,t_70)

Main parts of EVM：

![在这里插入图片描述](https://img-blog.csdnimg.cn/96db5d3f65294eeaa6120794a7eaeae5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l6cGJyaWdodA==,size_16,color_FFFFFF,t_70)


**EVM Code**

EVM code is Ethereum Virtual Machine code, which refers to the code of programming language that EVM can contain. The EVM code associated with the account is executed every time a message is sent to the account, and has the ability to read / write storage and send messages by itself.

**Storage**

Storage is a persistent storage space that can be read, written and modified. It is also a place where each contract can persistently stores data. Storage is a huge map with 2 ^ 256 slots and 32bytes in one slot.

**Mchine State**

Mchine State is where evm code is executed, contains program counter, stack, memory.

# Implementation of EVM on Substrate

The compatibility of Ethereum on substrate is mainly through the use of  Frontier layer. Frontier is developed by parity, it is an Ethereum compatible layer on substrate, which enables the substrate based chain to run unmodified Ethereum contracts. Frontier is still under development and mainly includes the following modules:


**Web3 RPC module**: existing tools and applications interact with Ethereum through Web3 RPC. When deploys Web3 RPC, existing tools and applications can be connected to the substrate-based node. For these tools and applications, they are just connected to another Ethereum network. For example, by simply configuring MetaMask , you can make MetaMask point to a node based on substrate, and then users can use MetaMask as usual. For MetaMask , it is only talking with Web3 RPC or API on substrate-based node.


**Ethereum module**: simulates how Ethereum works, including blocks, receipts, logs, and being able to subscribe to log events.


**Complete EVM implementation**: EVM is the contract virtual machine of Ethereum. Frontier integrates EVM module to be compatible with EVM on Ethereum.

# Compare With Ethereum Network
The Frontier should be able to produce nearly identical results compared to the Ethereum mainnet, including gas cost and balance changes.
Observable differences include:
1.The available length of block hashes may not be 256 depending on the configuration of the System module in the Substrate runtime.
2.Difficulty and coinbase, which do not make sense in this module and is currently hard coded to zero.




# Summary

Frontier is the key functional module of substrate. The fully functional Frontier means that all substrate-based chains (as well as Polkadot or Kusama parallel chains) will be able to provide full Ethereum compatibility for its developer community!
