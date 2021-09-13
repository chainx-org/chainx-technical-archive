# The Design and Implementation of pallet-coming-auction

## 1. Intro

The current NFT market continues to be active. In order to meet the growing demand for Coming NFT trading, it is necessary to provide the Coming NFT exchange function. There are two ways to realize the NFT exchange function-NFT auction contract and NFT exchange contract. This article introduces the coming nft Dutch auction Design and implementation of contracts.

## 2. Dutch auction

A Dutch auction is a type of auction where the auctioneer begins with a high asking price which is lowered until some participant is willing to accept the auctioneer's price, or a predetermined reserve price (the seller's minimum acceptable price) is reached. The winning participant pays the last announced price. This is also known as a "clock auction" or an open-outcry descending-price auction.

## 3. Pallet-coming-auction


![image](https://user-images.githubusercontent.com/8869892/132611008-4b39b11c-51f7-4d21-9707-4b59ceb1a59a.png)


![image](https://user-images.githubusercontent.com/8869892/132611596-f7704a24-97dc-4b94-94ef-d869ef7a49dd.png)


- core struct
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

- core storage

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

- core fuctions
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

- current price of cid

It is determined when creating a new auction: `start_price`, `end_price`, `duration`, `start`, set the current blockheight=h, and the price of cid as `p`
So:
```
 (1) When h-start <duration: p = start_price + (h-start) * (end_price-start_price) * (h-start) / duration
 (2) When h-start >= duration: p = end_price
 (3) When cid is not being auctioned: p = 0
```

- *Duration is at least 100 (10 minutes to generate 100 blocks)*
- *Usually end_price < start_price, which belongs to Dutch auction; allows end_price> start_price, which belongs to British auction; when end_price=start_price, it belongs to regular price sale*


## 4. NFT-Auction vs NFT-Swap

The same point is that NFT holders exchange ownership of NFT and Token with Token holders.

The difference is that the NFT quotation in the NFT auction contract is a monotonic function of time, that is, the selling price of the NFT during the auction increases or decreases (increase is a British auction, and decrease is a Dutch auction. Decrease is obviously more common), as long as someone bids more than It is equal to the quotation, and the auction is completed and ended; while the quotation in the NFT exchange contract is a constant, that is, the NFT selling price does not change until someone bids and the selling price is the same, and the exchange transaction ends.

When the NFT selling price in the NFT auction contract does not change, it is equivalent to the NFT exchange contract at this time.


