#  Ethereum account transfer to Substrate account by precompiled contract

## 1 Introduction
Through the previous article "How Does Frontier Manages Substrate Accounts and Ethereum Accounts", we know that frontier is an ethereum compatibility layer based on substrate. At the same time, we also understand the generation rules of substrate accounts and ethereum accounts. This article introduces the use of precompile contract, to solve the problem of transferring funds from the ethereum account to the substrate account mentioned in the previous article.

## 2 Review

![image](https://user-images.githubusercontent.com/8869892/131976203-2b713234-6aea-4a55-b888-14d68e8f5727.png)

As shown in the figure, the transfer  between `AccountId_substrate` and `AccountId_ethereum`
- [x] `AccountId_substrate ->  AccountId_substrate`:  substrate api (sr25519 signed tx), path-1 in the diagram
- [x] `AccountId_substrate ->  AccountId_ethereum`:  substrate api (sr25519 signed tx), path-1 in the diagram
- [x] `AccountId_ethereum -> AccountId_ethereum`:  eth api (ecdsa signed tx),  path-2 in the diagram
- [ ] `AccountId_ethereum -> AccountId_substrate`: The ecdsa account provided by substrate is not compatible with the ethereum account. Although frontier provides the path 3  in the diagram (the withdraw method of pallet-evm) , it is not universal. You need to customize the `Ecdsa Signature` to be compatible with the ethereum account, such as Moonbeam's solution. Paths `3`,`4`, and `5` in the figure are all possible technical paths.

This article introduces an implementation of path 5: call the evm contract through the eth api to complete the verification of the ethereum account, and then call the precompile contract inside the evm contract, and the implementation of precompile is to directly call the pallet-balances transfer.

## 3 Precompiled contracts

Precompiled contracts are used in the EVM to provide more complex library functions (usually used for complex operations such as encryption, hashing, etc.) that are not suitable for writing in opcode. They are applied to contracts that are simple but frequently called, or that are logically fixed but computationally intensive. Precompiled contracts are implemented on the client-side with client code, and because they do not require the EVM, they run fast. 

In frontier's EVM implementation, the EVM can call WASM contracts through pre-compiled contracts.

In this article, we call the transfer function of pallet-balances through the written solidity contract.

Below is the code snippet of our pre-compiled contract running on SherpaX:

```rust
pub struct SherpaxPrecompiles<R>(PhantomData<R>);

impl<R> PrecompileSet for SherpaxPrecompiles<R>
where
    R: pallet_evm::Config,
    R::Call: Dispatchable<PostInfo = PostDispatchInfo> + GetDispatchInfo + Decode,
    <R::Call as Dispatchable>::Origin: From<Option<R::AccountId>>,
{
    fn execute(
        address: H160,
        input: &[u8],
        target_gas: Option<u64>,
        context: &Context,
    ) -> Option<Result<PrecompileOutput, ExitError>> {
        match address {
            // Ethereum precompiles
            a if a == hash(1) => Some(ECRecover::execute(input, target_gas, context)),
            a if a == hash(2) => Some(Sha256::execute(input, target_gas, context)),
            a if a == hash(3) => Some(Ripemd160::execute(input, target_gas, context)),
            a if a == hash(4) => Some(Identity::execute(input, target_gas, context)),
            a if a == hash(5) => Some(Modexp::execute(input, target_gas, context)),
            a if a == hash(6) => Some(Bn128Add::execute(input, target_gas, context)),
            a if a == hash(7) => Some(Bn128Mul::execute(input, target_gas, context)),
            a if a == hash(8) => Some(Bn128Pairing::execute(input, target_gas, context)),
            // Non Ethereum precompiles
            a if a == hash(1024) => Some(Dispatch::<R>::execute(input, target_gas, context)),
            a if a == hash(1025) => Some(crate::withdraw::Withdraw::<R>::execute(input, target_gas, context)),
            _ => None,
        }
    }
}
```

The `0x401`(eq 1025) is the withdraw precompiled contract address.

The composition of input:  `input = from(evm address, 20 bytes) + to(substrate pubkey, 32 bytes) + value(32 bytes)`

```rust
pub struct Withdraw<T: pallet_evm::Config> {
    _marker: PhantomData<T>,
}

impl<T> Precompile for Withdraw<T>
    where
        T: pallet_evm::Config,
        T::AccountId: Decode,
{
    fn execute(
        input: &[u8],
        _target_gas: Option<u64>,
        context: &Context,
    ) -> core::result::Result<PrecompileOutput, ExitError> {
        log::debug!(target: "evm", "withdraw: input: {:?}", input);
        log::debug!(target: "evm", "withdraw: caller: {:?}", context.caller);

        const BASE_GAS_COST: u64 = 45_000;

        // input = from(evm address, 20 bytes) + to(substrate pubkey, 32 bytes) + value(32 bytes)
        if input.len() != 20 + 32 + 32 {
            return Err(ExitError::Other("invalid input".into()))
        }

        let from = H160::from_slice(&input[0..20]);
        log::debug!(target: "evm", "withdraw: from: {:?}", from);

        let address_account_id = T::AddressMapping::into_account_id(from);
        log::debug!(target: "evm", "withdraw: source: {:?}", address_account_id);

        let mut target = [0u8; 32];
        target[0..32].copy_from_slice(&input[20..52]);
        let dest = T::AccountId::decode(&mut &AccountId32::new(target).encode()[..])
            .map_err(|_| ExitError::Other("decode failed".into()))?;
        log::debug!(target: "evm", "withdraw: target: {:?}", HexDisplay::from(&target));

        let value = U256::from_big_endian(&input[52..84])
            .low_u128().unique_saturated_into();
        log::debug!(target: "evm", "withdraw: value: {:?}", value);

        T::Currency::transfer(
            &address_account_id,
            &dest,
            value,
            ExistenceRequirement::AllowDeath,
        ).map_err(|err| {
            log::debug!(target: "evm", "withdraw: err = {:?}", err);

            ExitError::OutOfFund
        })?;

        log::debug!(target: "evm", "withdraw: success");

        Ok(PrecompileOutput {
            exit_status: ExitSucceed::Stopped,
            cost: BASE_GAS_COST,
            output: Default::default(),
            logs: Default::default(),
        })
    }
}
```


## 4 Solidity contract

The contract for transferring funds from the ethereum account to the substrate account,

`to` is of type bytes32, which is the public key of the recipient, 
`value` is the amount of  transfer

```solidy
pragma solidity ^0.7.0;

contract WithdrawPrecompile {
     event Withdraw(address indexed from, bytes32 indexed to, uint256 value);

     // evm account transfer ksx to substrate account
     // @to is substrate account public key
     // @value is amount of balance
     function withdraw(
        bytes32 to,
        uint256 value
    ) public returns (bool) {
        bytes memory input = abi.encodePacked(msg.sender, to, value);

        // Dynamic arrays will add the array size to the front of the array, so need extra 32 bytes.
        uint len = input.length;

        assembly {
            if iszero(
                // from(20 bytes) + to(32 bytes) + value(32 bytes)
                staticcall(gas(), 0x401, add(input, 0x20), len, 0x00, 0x00)
            ) {
                revert(0, 0)
            }
        }

        emit Withdraw(msg.sender, to, value);

        return true;
    }
}
```

## 5 SherpaX transfer demo

- solidity contract address: `0xc01Ee7f10EA4aF4673cFff62710E1D7792aBa8f3`
- ethereum account: `0xf24FF3a9CF04c71Dbc94D0b566f7A27B94566cac`
- substrate account:  `Fr4NzY1udSFFLzb2R3qxVQkwz9cZraWkyfH4h3mVVk7BK7P`,  it's public key `0x90b5ab205c6974c9ea841be688864633dc9ca8a357843eeacf2314649965fe22`,  balance is 0.
- transfer balance: 1000000000000000000 wei =  1 KSX

expect balance after transfer: 1000000000000000000 wei =  1 KSX

![image](https://user-images.githubusercontent.com/8869892/131990239-49d17073-63fd-4c10-8427-8af2fae66f27.png)

![image](https://user-images.githubusercontent.com/8869892/131985687-986b73db-bb9f-41e1-adfa-72be89ac775e.png)

![image](https://user-images.githubusercontent.com/8869892/131985808-cdcfa7c3-3d7e-4b4d-bf72-b6c7adc078f8.png)


