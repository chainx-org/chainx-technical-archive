# A design of talent contract NFT on the chain

## 1. Introduction
In the future, there may be the following scenarios:
A certain entertainment star S signs a contract with a certain brokerage company A, and at the same time the two parties agree to mint a talent contract NFT on MinixChain (the ownership belongs to both parties), which means that the star S belongs to the brokerage company A. When the star S or the brokerage company A finds a new broker  company B who can cooperate, they can transfer the ownership of the NFT to both the star S and the new brokerage company B, which also means that the star S and the brokerage company A terminate the contract and sign with the brokerage company B.

## 2. Requirement analysis
The fundamental characteristics of NFT are ownership transferability and uniqueness.
We remove redundant information and try to find a feasible solution:

Defined:
- Party A (party_a) is the talent party to be signed, Party B (party_b) is the brokerage company to be signed, and the new Party B (new_party_b) is the role that appears while transfering the talent contract NFT on the chain.
- Party A and Party B establish contact through encrypted terminals and leave key information for signing on the chain.
- Since the information on the chain is public, in order to ensure that the signing process of both parties is not damaged by a third party and can be revoked,We use Hash-Time-Lock technology.

## 3. WorkFlow
![talent-contract1](https://user-images.githubusercontent.com/8869892/126440467-af13bed5-f5d2-41a8-b416-eee3e2d126b3.png)

![talent-contract3](https://user-images.githubusercontent.com/8869892/126441819-0ea8f0ea-7d2c-42c4-807b-bc1cb18f8d88.png)

## 4. Detailed design
### 4.0 Core data structure

```rust
#[derive(Clone, Eq, PartialEq, Encode, Decode)]
pub enum Status<AccountId> {
    // (secret_hash)
    WaitingToSign(H256),
    // (confirmer, new_party_b)
    WaitingToConfirm(Option<AccountId>, AccountId),
    CanTransfer,
}

#[derive(Clone, Eq, PartialEq, Encode, Decode)]
pub struct Contract<AccountId, BlockNumber> {
    pub digest: H256,
    pub status: Status<AccountId>,
    pub updated: BlockNumber,
    pub party_a: Option<AccountId>,
    pub party_b: Option<AccountId>,
}
```

### 4.1. Signing(Minting NFT) process (contract drafter and contract signer): draft, sign
#### (a) draft

Either of both party_a and party_b can `draft` a talent contract on the chain.

Preconditions: 
  - The contract drafter generates the 32-bytes digest of Off-Chain-Contract file.
  - The contract drafter generates a hash lock (blake2_256 hash algorithm) : (secret, secret_hash)

`draft(origin, digest, secret_hash, is_party_a)`

Parameters:
  - origin: the contract drafter
  - digest: 32-bytes file hash
  - secret_hash: the hash lock set by the drafter of the contract
  - is_party_a: Whether the drafter of the contract is Party A or Party B.

`Status`: `WaitingToSign`

#### (b) sign
Party A and Party B establish a connection through an encrypted terminal, and the drafter of the contract passes the `secret` to the final signer of the contract in an encrypted environment

Preconditions: 
  - Current status is `WaitingToSign`
  - The contract signer got the `secret`

`sign(origin, digest, secret)`

Parameter:
 - origin:  the contract signer
 - digest: 32-bytes file hash
 - secret: hash lock original text

`Status`: `Cantransfer`

### 4.2 Transfering(Transfer NFT) process (contract initiator, contract confirmer): transfer, confirm
The owner of the contract on the chain belongs to both parties,
Both parties can initiate a transfer to change the party B of the on-chain contract, and then the other party will confirm and complete the transfer of the on-chain contract.

#### (1) transfer
Party A or Party B establishes contact with the new Party B through an encrypted terminal

Preconditions: 
  - Current status is `CanTransfer`

`transfer(origin, digest, new_party_b)`

Parameter:
  - origin: contract initiator of transfering
  - digest:  32-bytes file hash
  - new_party_b: new `Party B`

`Status`: `WaitingToConfirm`

#### (2) confirm
The owner of the contract on the chain belongs to both parties, 
so one party needs to initiate and the other party confirms to complete the transfer operation of the contract on the chain.

Preconditions:
  - Current status is `WaitingToConfirm`
  - Correct confirmer

`confirm(origin, digest)`

Parameter:
origin: contract confirmer of transfering
digest: 32-bytes file hash

### 4.3. revoke signing and transfering process
#### revoke
The validity period is 1 hour. 
During the validity period, it can only be revoked by Party A or Party B of the on-chain contract.
After expiration, it can be revoked by anyone.

- After `draft`, the status of the on-chain contract is `WaitingToSign`, after revoke,  the contract will be removed.

`Status`: `none`

- After `transfer`, the status of the contract on the chain is `WaitingToConfirm`, after revoke, the status of the contract on the chain will rollback to `CanTransfer`

`Status`: `CanTransfer`


