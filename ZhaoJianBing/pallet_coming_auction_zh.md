# pallet-coming-auction 拍卖合约设计与实现

## 1. 简介
当前NFT市场持续活跃, 为满足日益增长的Coming NFT交易需求, 需要提供Coming NFT交换功能. 实现NFT交换功能有2种方式——NFT拍卖合约和NFT交易所合约. 本文介绍了coming nft荷兰式拍卖合约的设计与实现.

## 2. 荷兰式拍卖
荷兰式拍卖是拍卖方式的一种. 拍卖师以高要价开始, 然后降低直到某些参与者愿意接受拍卖师的价格, 或达到预定的底价(卖方的最低可接受价格). 获胜的参与者支付最后公布的价格. 这也称为"时钟拍卖(clock auction)"或公开喊价降价拍卖。

## 3. 拍卖合约各个模块

![image](https://user-images.githubusercontent.com/8869892/132611008-4b39b11c-51f7-4d21-9707-4b59ceb1a59a.png)


![image](https://user-images.githubusercontent.com/8869892/132611596-f7704a24-97dc-4b94-94ef-d869ef7a49dd.png)


- 核心数据结构
```rust
#[derive(Clone, Eq, PartialEq, Encode, Decode)]
#[cfg_attr(feature = "std", derive(Debug, Serialize, Deserialize))]
#[cfg_attr(feature = "std", serde(rename_all = "camelCase"))]
pub struct Auction<AccountId, Balance, BlockNumber> {
    pub seller: AccountId,
    pub start_price: Balance,
    pub end_price: Balance,
    pub duration: BlockNumber,
    pub start: BlockNumber,
}
```

- 核心存储结构

```rust
    /// The auctions in progress
    #[pallet::storage]
    #[pallet::getter(fn auctions)]
    pub type Auctions<T: Config> =
    StorageMap<_, Twox64Concat, Cid, Auction<T::AccountId, BalanceOf<T>, T::BlockNumber>>;

    /// The auction stats.
    /// (total_auctions, success_auctions, cancel_auctions)
    #[pallet::storage]
    #[pallet::getter(fn stats)]
    pub(super) type Stats<T: Config> = StorageValue<_, (u64, u64, u64), ValueQuery>;

    /// The pallet admin key.
    #[pallet::storage]
    #[pallet::getter(fn admin_key)]
    pub(super) type Admin<T: Config> = StorageValue<_, T::AccountId>;

    /// The protocol fee point.
    #[pallet::storage]
    #[pallet::getter(fn point)]
    pub(super) type Point<T: Config> = StorageValue<_, u8, ValueQuery>;

    /// The emergency stop.
    #[pallet::storage]
    #[pallet::getter(fn in_emergency)]
    pub(super) type InEmergency<T: Config> = StorageValue<_, bool, ValueQuery>;
```

- 核心函数
```rust
pub fn create(
  origin: OriginFor<T>, 
  cid: Cid, 
  start_price: BalanceOf<T>,
  end_price: BalanceOf<T>, 
  duration: T::BlockNumber
) -> DispatchResult

pub fn bid(
  origin: OriginFor<T>,
  cid: Cid,
  value: BalanceOf<T>,
) -> DispatchResult

pub fn cancel(
  origin: OriginFor<T>,
  cid: Cid,
) -> DispatchResult

pub fn pause(
  origin: OriginFor<T>
) -> DispatchResult 

pub fn unpause(
  origin: OriginFor<T>
) -> DispatchResult

pub fn cancel_when_pause(
  origin: OriginFor<T>,
  cid: Cid
) -> DispatchResult

pub fn set_fee_point(
  origin: OriginFor<T>,
  new_point: u8
) -> DispatchResult

pub fn set_admin(
  origin: OriginFor<T>,
  new_admin: <T::Lookup as StaticLookup>::Source,
) -> DispatchResult
```

- cid NFT 当前价格

创建新的auction时已确定: `start_price`, `end_price`, `duration`, `start`,设当前blockheight=h,  cid的价格为p
那么:
``` 
 (1) 当 h - start < duration时:   p = start_price + (h - start) * (end_price - start_price) * (h - start) / duration
 (2) 当 h - start >= duration时:  p = end_price
 (3) 当 cid 没有正在拍卖:  p = 0
```

- *duration最少是100(产生100个block是10分钟)*
- *通常end_price < start_price, 属于荷兰式拍卖; 允许end_price > start_price, 属于英式拍卖; 当end_price=start_price时, 属于常价售卖*


## 4. NFT拍卖合约 vs NFT交换合约

相同点都是NFT的持有者与Token持有者进行NFT和Token的所有权交换.

不同的是, NFT拍卖合约中的NFT报价是时间的单调函数, 即拍卖期间NFT的卖价递增或递减(递增是英式拍卖,递减是荷兰式拍卖.递减显然更常见), 只要有人出价大于等于报价, 拍卖成交并结束; 而NFT交换合约中报价是一个常数, 即NFT的卖价不变, 直到有人出价和卖价相同, 交换成交并结束.

当NFT的拍卖合约中NFT卖价不变时, 此时与NFT交换合约等效.

