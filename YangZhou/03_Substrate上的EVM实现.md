## 简介

​       比特币是目前最著名的最大的分布式账本。相比而言，以太坊不仅有自己的原生加密货币，同时它也实现了更强大的功能：智能合约。对于这个更复杂的特性，需要进行更复杂的类比。以太坊不是分布式账本，而是分布式状态机。以太坊的状态是一个大型数据结构，它不仅保存所有账户和余额，而且还保存了一个机器状态，可以根据预定义的一组规则从一个区块到另一个区块进行更改，并且可以执行任意机器代码。改变状态的具体规则由EVM定义。

## EVM的整体原理

​       以太坊虚拟机是以太坊智能合约的运行时环境。它不仅是沙盒封装的，而且实际上是完全隔离的，这意味着在EVM中运行的代码无法访问网络、文件系统和其他进程。甚至智能合约之间的访问也是受限的。

​      以太坊底层通过EVM模块支持合约的执行与调用，调用时根据合约地址获取到合约代码，载入到EVM中运行。通常智能合约的开发流程是用solidlity编写逻辑代码，再通过编译器编译成字节码，最后再发布到以太坊上。


![在这里插入图片描述](https://img-blog.csdnimg.cn/59ec48be1ff14530841d6f59e338115e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l6cGJyaWdodA==,size_16,color_FFFFFF,t_70)



EVM的主要部分：

![在这里插入图片描述](https://img-blog.csdnimg.cn/96db5d3f65294eeaa6120794a7eaeae5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l6cGJyaWdodA==,size_16,color_FFFFFF,t_70)

**EVM code**

EVM代码是以太坊虚拟机代码， 指以太坊可以包含的编程语言的代码。与帐户相关联的EVM 代码在每次消息被发到这个账户的时候被执行，并且具有读/写存储和自身发送消息的能力。

 

**Mchine State**

Mchine State是执行evm代码的地方，包含程序计数器、堆栈和内存。

 

**Storage**

Storage是一个可读、可写、可修改的持久存储的空间，也是每个合约持久化存储数据的地方。Storage是一个巨大的map，一共有2^256个插槽，每个插糟有32byte。



## Substrate上EVM的实现



Substrate上对以太坊的兼容主要通过使用 Frontier 层来实现。Frontier 由 Parity开发，它是 Substrate 上的以太坊兼容层，能让基于 Substrate 的链运行未经修改的以太坊合约。Frontier 目前还在开发中，主要包括以下几个模块：

**Web3 RPC 模块**： 现有的工具和应用程序就是通过 Web3 RPC 与以太坊交互的，Moonbeam 部署了 Web3 RPC，就可以让现有的工具和应用连接到 Moonbeam，而对于这些工具和应用来说，就像只是连接到了另一个以太坊网络一样。举个例子，只需要对 MetaMask 进行简单的配置，就可以让 MetaMask 指向一个基于 Moonbeam 的节点，然后用户就可以正常地像平时一样使用 MetaMask，而对于 MetaMask 来说它只是在和 Moonbeam 上的 Web3 RPC 或 API 对话。

**Ethereum 模块**： 模拟了以太坊如何工作，包括区块、收据、日志、能够订阅日志事件等。

**完整的 EVM 实现**： EVM 是以太坊的合约虚拟机，Moonbeam 集成了 EVM 模块，从而兼容以太坊上的 EVM。



## 以太坊网络的对比



与以太坊主网络相比，Frontier几乎能够产生相同的结果，包括gas花费以及余额的变化。

 较明显的差异包括：

1.根据Substrate 运行时中系统模块的配置，区块散列的可用长度可能不是256。

2.Difficulty 和coinbase在Frontier模块中没有意义，目前硬编码为零。



## 总结

​		Frontier是 substrate的重点功能模块。功能齐全的 Frontier 意味着所有 Substrate 链（以及 Polkadot 或 Kusama 平行链）将能够为其开发者人群提供完全的以太坊兼容性！

