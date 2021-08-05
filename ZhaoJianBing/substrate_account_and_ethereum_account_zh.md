# frontier是如何管理substrate账户和ethereum账户的

## 1. 简介
[frontier](https://github.com/paritytech/frontier) 是基于substrate实现的ethereum兼容层,可以将ethereum上的部署的dapp平滑的迁移到substrate上,且frontier完全兼容ethereum的api. 当eth交易通过frontier提供的 eth api接口执行交易时, ethereum 账户(或地址)是如何在substrate上保存的? substrate账户和ethereum账户能互相转账吗?

## 2. frontier的架构

frontier 的实现主要由以下组件构成:

- runtime: 
  - evm: 引用的是 [SputnikVM](https://github.com/rust-blockchain/evm) EVM引擎
  - pallet-evm: 封装了substrate环境下的上下文, 比如账户的余额的处理
  - pallet-ethereum: 存储eth交易及其收据, 将eth交易转换成substrate交易
- rpc module: ethApi, ethFilterApi, netApi, ethPubSubApi, web3Api

![image](https://user-images.githubusercontent.com/8869892/128314802-21f85c2b-a956-4f1e-9732-78d5ce7ad0b8.png)

如图所示, substrate账户和ethereum账户共用了pallet-balances, 也就是说ethereum账户被映射为substrate账户存在了substrate链上. 

`pallet-evm` 提供了 `AddressMapping` 关联类型, 可自定义ethereum账户转为substrate账户的方法.

```
/// EVM module trait
pub trait Config: frame_system::Config + pallet_timestamp::Config {
	/// Calculator for current gas price.
	type FeeCalculator: FeeCalculator;

	/// Maps Ethereum gas to Substrate weight.
	type GasWeightMapping: GasWeightMapping;

	/// Block number to block hash.
	type BlockHashMapping: BlockHashMapping;

	/// Allow the origin to call on behalf of given address.
	type CallOrigin: EnsureAddressOrigin<Self::Origin>;
	/// Allow the origin to withdraw on behalf of given address.
	type WithdrawOrigin: EnsureAddressOrigin<Self::Origin, Success=Self::AccountId>;

	/// Mapping from address to account id.
	type AddressMapping: AddressMapping<Self::AccountId>;
	/// Currency type for withdraw and balance storage.
	type Currency: Currency<Self::AccountId>;

	/// The overarching event type.
	type Event: From<Event<Self>> + Into<<Self as frame_system::Config>::Event>;
	/// Precompiles associated with this EVM engine.
	type Precompiles: PrecompileSet;
	/// Chain ID of EVM.
	type ChainId: Get<u64>;
	/// The block gas limit. Can be a simple constant, or an adjustment algorithm in another pallet.
	type BlockGasLimit: Get<U256>;
	/// EVM execution runner.
	type Runner: Runner<Self>;

	/// To handle fee deduction for EVM transactions. An example is this pallet being used by `pallet_ethereum`
	/// where the chain implementing `pallet_ethereum` should be able to configure what happens to the fees
	/// Similar to `OnChargeTransaction` of `pallet_transaction_payment`
	type OnChargeTransaction: OnChargeEVMTransaction<Self>;

	/// Find author for the current block.
	type FindAuthor: FindAuthor<H160>;

	/// EVM config used in the module.
	fn config() -> &'static EvmConfig {
		&ISTANBUL_CONFIG
	}
}
```

## 3. substrate账户和ethereum账户的生成
先看ethereum账户生成的过程: `ecdsa-secp256k1 私钥` -> `ecdsa-secp256k1 公钥` -> `ethereum AccountId20` -> `Hex格式address`

- 第一步:  私钥(private key)
　伪随机数产生的256bit私钥示例(256bit  16进制32字节)

  ```
  18e14a7b6a307f426a94f8114701e7c8e774e7f9a47e2c2035db29a206321725
  ```

- 第二步: 公钥(public key)
   
   - 1.采用椭圆曲线数字签名算法ECDSA-secp256k1将私钥(32字节)映射成公钥(65字节)(前缀04+X公钥+Y公钥):

  ```
  04
  50863ad64a87ae8a2fe83c1af1a8403cb53f53e486d8511dad8a04887e5b2352
  2cd470243453a299fa9e77237716103abc11a1df38855ed6f2ee187e9c582ba6
  ```
    
    - 2.跳过第一个字节(前缀`04`), 拿剩下的64字节公钥(非压缩公钥)来hash，计算公钥的 Keccak-256 哈希值(32bytes):

  ```
  fc12ad814631ba689f7abe671016f75c54c607f082ae6b0881fac0abeda21781
  ```

    - 3.取上一步结果的后20bytes即链上存储的`AccountId20`:

  ```
  1016f75c54c607f082ae6b0881fac0abeda21781
  ```

- 第三步: `AccountId20`的Hex格式化,即为ethereum address: 

  ```
  0x1016f75c54c607f082ae6b0881fac0abeda21781
  ```

与之类似, substrate账户生成的过程: `sr25519 私钥` -> `sr25519 公钥` -> `substrate AccountId32` -> `Base58格式address`
- 第一步:  私钥(private key)　

- 第二步: 公钥(public key): sr25519私钥生成32字节的公钥, 即为链上存储的 `AccountId32`

- 第三步: `AccountId32`的Base58格式化, 即为 substrate address.


## 3. substrate账户和ethereum账户之间的转账

上面我们已经知道ethereum账户通过自定义方法转换成substrate账户,共用pallet-balances.

![image](https://user-images.githubusercontent.com/8869892/128326450-9efdc8b3-0e33-424f-838b-9c8ad5816c79.png)


如图所示, `AccountId_substrate`和`AccountId_ethereum` 之间的转账操作
- [x] `AccountId_substrate ->  AccountId_substrate`:  substrate api (sr25519 signed tx), 图中的路径1
- [x] `AccountId_substrate ->  AccountId_ethereum`:  substrate api (sr25519 signed tx), 图中的路径1
- [x] `AccountId_ethereum -> AccountId_ethereum`:  eth api (ecdsa signed tx), 图中的路径2
- [ ] `AccountId_ethereum -> AccountId_substrate`: substrate提供ecdsa账户和ethereum账户并不兼容, 虽然frontier提供了图中路径3的方式(pallet-evm withdraw), 但并不通用.需要自定义`Ecdsa Signature`兼容ethereum账户, 比如Moonbeam的解决方案.

## 4. Minix 如何处理ethereum账户到substrate账户的转账
上一节的图中, 提供了路径3和路径4两种方案, 来处理 `AccountId_ethereum -> AccountId_substrate` 

- 路径3: 结合自定义ethereum账户转为substrate账户的方法, 修改minix的MultiSignature兼容ethereum账户, 通过pallet-evm的withdraw方法完成ethereum账户到substrate账户的转账.

- 路径4: 自定义一个无签名的`withdraw_ecdsa`方法, 通过substrate的Call, 组装transfer调用, 用ecdsa私钥签名, `withdraw_ecdsa` 完成路径四的有效性检查和ethereum账户到substrate账户的转账.

除此之外, minix还需要处理AccountId_substrate balance 与 AccountId_ethereum balance之间的转换.

minix使用8位tokenDecimals,  而ethereum使用的是18位tokenDecimals

方案一:  将minix的tokenDecimals 升级为18位.
方案二:  mini总量无上限, 关闭 `AccountId_ethereum -> AccountId_substrate` 转账, 降低影响.

