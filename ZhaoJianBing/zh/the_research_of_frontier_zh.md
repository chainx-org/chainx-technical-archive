# 调研frontier
## 1. 简介
以太坊的影响力越来越大,作为新兴的区块链开发框架,Substrate不得不考虑如何兼容EVM Dapp.虽然Parity提供了Bridge的方式将Ethereum和Polkadot两个区块链世界连接起来,但目前也仅仅是资产的跨链转移或消息的跨链转移.为了支持Ethereum Dapp的低成本迁移至Substrate, Parity在2018年底就启动了[frontier](https://github.com/paritytech/frontier)项目.

## 2. 以太坊兼容层
frontier 是在substrate上构建了以太坊兼容层, 其目标是:
- 通过兼容层运行普通的Ethereum Dapp.
- 能导入以太坊主网的状态

Ethereum Dapp主要有三种类型:
- 不处理私钥的Dapps: 他们依赖eth_sendTransaction并期望 RPC 节点来处理私钥.
- 带有私钥的Dapps: 这些 dapps 在内部处理私钥, 离线签名交易并使用eth_sendRawTransaction发送交易.
- 专用Dapps: 具有特殊功能的 dapps. 其中包括严重依赖以太坊内部实现细节的那些Dapp. 一个例子就是去中心化的矿池合约.

frontier 只支持前两种, 而忽略了专用 dapp 的情况, 主要是因为模拟以太坊实现的微妙细节是昂贵的.

frontier 的实现主要由以下组件构成:

- runtime: 
  - evm: 引用的是 [SputnikVM](https://github.com/rust-blockchain/evm) EVM引擎
  - pallet-evm: 封装了substrate环境下的上下文, 比如账户的余额的处理
  - pallet-ethereum: 存储eth交易及其收据, 将eth交易转换成substrate交易
- rpc module: ethApi, ethFilterApi, netApi, ethPubSubApi, web3Api

![image](https://user-images.githubusercontent.com/8869892/127769409-30f02364-3db7-488b-b91c-c171d4c73b5a.png)

一笔普通转账交易的处理过程:
- Ethereum Dapp 交易通过eth_sendTransaction或eth_sendRawTransaction rpc 发送到  pallet-ethereum, 
- pallet-ethereum 将交易转换成 substrate 无签名交易 并调用 pallet-evm
- pallet-evm 检查交易合法性(如账户余额)并执行evm
- evm 调用 pallet-balances 完成转账.

## 3. 当前进展
frontier项目虽然启动较早, 但目前仍处于POC阶段.
当前polkadot社区, 支持EVM的项目有如下几个:
- moonbeam:
  moonbeam 完全在frontier的基础上推进etheteum dapp迁移至substrate, 其账户体系在设计之初就采用与ethereum相同的ecdsa 签名算法, 包括balances的decimal也和eth相同. 其官方文档较全, 开发活跃.
- plasm:
  plasm也是基于frontier做的EVM支持,基本没有对frontier做删改.
- acala
  跳过了frontier繁重的ethereum支持,直接基于SputnikVM自己做evm的支持. 其目标仅仅是支持evm合约的部署和调用, 并非是兼容eth的api.

## 4. minix移植evm目标与难点
目标:
- 通过移植frontier的evm方案, 运行普通的ethereum dapp.
难点:
(1) evm合约执行的调试
(2) 手续费的收取(minix 8位decimal与 eth 18位decimal的转换)
