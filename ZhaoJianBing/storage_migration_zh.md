1. 简介
存储迁移发生在runtime升级的过程中. substrate pallet开发中需要存储私有数据，这些数据通常有StorageValue(单一键值存储), StorageMap(键值对存储), StorageDoubleMap(键值对存储, 两个键一个值). 当需要改变pallet中的存储(增删改)时, 就需要考虑存储迁移方案了.

2. 批量迁移和分散迁移
- 批量迁移: 主动性,一次完成迁移. substrate官方提供的存储迁移方案, 将所有需要迁移的存储放在一个批处理操作中, 由runtime upgrade触发, 特点是, 去中心化程度较低(需要runtime升级权限), 当数据量较大时, 迁移过程较长, 以至于要延长出块时间来完成迁移.

- 分散迁移: 被动性,多次完成迁移. 考虑到数据量较大时, 批量迁移过程时间较长, 不确定风险存在.将存储迁移的触发条件设置为"Modify On Access", 这样做的特点是去中心化程度较高(由相关的用户完成操作), 整体上看迁移过程更长,链上可能长期存在多个版本的数据, 有较大的维护成本.

3. 本文着重介绍Substrate推荐的方式——批量迁移. 其存储迁移方案实施步骤为:
- (1) 找到存储迁移方案的入口:
         重载on_runtime_upgrade
  ```
  #[pallet::hooks]
  impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
      fn on_runtime_upgrade() -> Weight {
          // impl your migration function
      }
  }
  ```

- (2) 实现存储迁移方案的migration函数
- (3) 确定存储迁移方案的触发条件
- (4) 升级runtime version
- (5) 编译wasm code 
- (6) 调用setcode或其他方式升级runtime

4. StorageKey
    StorageValue和StorageMap是最常用的存储形式,其底层是rocksdb.
- 根据下面的StorageKey计算方法, 我们可以计算出存储项的StorageKey.

  ```
  For storage values:
    xxhash128("ModuleName") + xxhash128("StorageName")
  For storage maps:
    xxhash128("ModuleName") + xxhash128("StorageName") + blake256hash("StorageItemKey")
  For storage double maps:
    xxhash128("ModuleName") + xxhash128("StorageName") + blake256hash("FirstKey") + blake256hash("SecondKey")
  ```

  通过StorageKey, 我们可以访问原来的数据.

  - 通过外部方式, 可参考
[Querying Substrate Storage via RPC](https://www.shawntabrizi.com/substrate/querying-substrate-storage-via-rpc/)
  - 通过内部方式
substrate 提供了两个storage相关的底层函数, get_storage_value返回的是字节数组
  ```
  use frame_support::storage::migration::{get_storage_value, put_storage_value}
  ```
   如果我们需要存储值与对应的结构体关联起来, 可以重新定义一个过渡的StorageValue或StorageMap, 或者使用StorageValue和StorageMap自带的translate和translate_value.

   如下示例(存储项Key在迁移后被删除) 
   
   原来的存储项定义如下:
   ModuleName = "ComingId", StorageName = "Key", 
  ```
  #[pallet::storage]
  #[pallet::getter(fn admin_key)]
  pub(super) type Key<T: Config> = StorageValue<_, T::AccountId, ValueQuery>;
  ```
  migration函数中定义了如下结构用来访问原来的存储项.
  ```
  pub struct __OldAdminKey<T>(sp_std::marker::PhantomData<T>);
  impl<T: Config> frame_support::traits::StorageInstance for __OldAdminKey<T> {
      fn pallet_prefix() -> &'static str {
          "ComingId"
      }
      const STORAGE_PREFIX: &'static str = "Key";
  }

  pub type OldAdminKey<T> =
  StorageValue<__OldAdminKey<T>, <T as frame_system::Config>::AccountId, ValueQuery>;
  ```

5. PalletVersion
- 每个pallet自带两个函数: current_version和storage_version
  - current_version: 当前的pallet version (与Cargo.toml中定义的相同).
  - storage_version: 上一次runtime upgrade后的pallet version. 每次runtime upgrade的时候, 会将当前的current_version存下.
  
  ```
  pub struct PalletVersion {
    /// The major version of the pallet.
    pub major: u16,
    /// The minor version of the pallet.
    pub minor: u8,
    /// The patch version of the pallet.
    pub patch: u8,
  }
  ```
  current_version和storage_version可以帮助我们更好的定义存储迁移的触发条件.
  

6. 注意要点
- 存储迁移的目标是旧数据的增删改和新数据的初始化.
- 旧数据的访问方式和新数据的访问方式.
- 迁移前后的pallet version.

7. 参考链接
- [OnRuntimeUpgrade](https://github.com/paritytech/substrate/blob/master/frame/support/procedural/src/pallet/expand/hooks.rs)
- [Substrate](https://github.com/paritytech/substrate)
- [Storage-Migrations](https://substrate.dev/substrate-how-to-guides/docs/storage-migrations/nicks-migration)
