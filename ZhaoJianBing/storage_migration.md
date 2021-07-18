1. Introduction
Storage migration occurs in the process of runtime upgrade. Private data needs to be stored in the development of substrate pallet. These data usually include StorageValue (single key-value storage), StorageMap (key-value pair storage), StorageDoubleMap (key-value pair storage,  two keys for one value). When you need to change the storage (addition, deletion, and modification) in the pallet, you need to consider the storage migration plan.

2. Batch migration and decentralized migration
- Batch migration: Proactive, complete migration at one time. The storage migration solution officially provided by substrate puts all the storage that needs to be migrated in a batch operation, which is triggered by runtime upgrade. It is characterized by a low degree of decentralization (requires runtime Upgrade permissions), when the amount of data is large, the migration process is longer, so that the block generation time must be extended to complete the migration.
- Decentralized migration: Passive, complete migration multiple times. Considering that when the amount of data is large, the batch migration process takes a long time and uncertain risks exist. Setting the trigger condition of storage migration to "Modify On Access", the characteristic of this is The degree of decentralization is high (operations are completed by related users), and the overall migration process is longer. There may be multiple versions of data on the chain for a long time, and there is a large maintenance cost.

3. This article focuses on the recommended method of Substrate-batch migration. The implementation steps of its storage migration plan are:
- (1) Find the entrance to the storage migration plan
Overload on_runtime_upgrade:
  ```
  #[pallet::hooks]
  impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
      fn on_runtime_upgrade() -> Weight {
          // impl your migration function
      }
  }
  ```
- (2) Implete the migration function of your  storage migration plan
- (3) Determine the trigger conditions of the storage migration plan
- (4) Upgrade runtime version
- (5) Compile wasm code
- (6) Call setcode or other methods to upgrade runtime

4. StorageKey
  StorageValue and StorageMap are the most commonly used storage forms, and the bottom layer is rocksdb.
  
- According to the following StorageKey calculation method, we can calculate the StorageKey of the storage item.

  ```
  For storage values:
    xxhash128("ModuleName") + xxhash128("StorageName")
  For storage maps:
    xxhash128("ModuleName") + xxhash128("StorageName") + blake256hash("StorageItemKey")
  For storage double maps:
    xxhash128("ModuleName") + xxhash128("StorageName") + blake256hash("FirstKey") + blake256hash("SecondKey")
  ```
  By StorageKey, we can access the original data.
  - External: [Querying Substrate Storage via RPC](https://www.shawntabrizi.com/substrate/querying-substrate-storage-via-rpc/)
  - Internal: Substrate provides two storage-related low-level functions, get_storage_value returns a byte array
  ```
  use frame_support::storage::migration::{get_storage_value, put_storage_value}
  ```  
  
  If we need to associate the stored value with the corresponding structure, we can redefine a transitional StorageValue or StorageMap, or use the translate and translate_value that come with StorageValue and StorageMap.

  The following example (the storage item Key is deleted after the migration).

  The old storage items are defined as follows:  ModuleName = "ComingId", StorageName = "Key"
  ```
  #[pallet::storage]
  #[pallet::getter(fn admin_key)]
  pub(super) type Key<T: Config> = StorageValue<_, T::AccountId, ValueQuery>;
  ```
  The following structure is defined in the migration function to access the old storage items.
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
- Each pallet comes with two functions: current_version and storage_version.
  - current_version: current pallet version (same as defined in Cargo.toml).
  - storage_version: The pallet version after the last runtime upgrade. Each time the runtime upgrade, the current current_version will be saved.

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
  `current_version` and `storage_version` can help us better define the trigger conditions for storage migration.

6. Notes
- The goal of storage migration is the addition, deletion, and modification of old data and the initialization of new data.
- How to access old data and how to access new data.
- Pallet version before and after migration.

7. Reference
- [OnRuntimeUpgrade](https://github.com/paritytech/substrate/blob/master/frame/support/procedural/src/pallet/expand/hooks.rs)
- [Substrate](https://github.com/paritytech/substrate)
- [Storage-Migrations](https://substrate.dev/substrate-how-to-guides/docs/storage-migrations/nicks-migration)
