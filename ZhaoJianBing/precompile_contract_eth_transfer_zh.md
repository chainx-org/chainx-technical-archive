# 通过预编译合约解决ethereum账户向substrate账户的转账问题

## 1. 简介
通过上一篇文章《frontier是如何管理substrate账户和ethereum账户的》, 我们知道了frontier是基于substrate实现的ethereum兼容层, 同时也了解了substrate账户和ethereum账户的生成规则,  本文介绍了通过precompile contract的方式, 解决上一篇文中所述的ethereum账户向substrate账户的转账问题。

## 2. 回顾ethereum账户向substrate账户转账问题

![image](https://user-images.githubusercontent.com/8869892/131976203-2b713234-6aea-4a55-b888-14d68e8f5727.png)

如图所示, `AccountId_substrate`和`AccountId_ethereum` 之间的转账操作
- [x] `AccountId_substrate ->  AccountId_substrate`:  substrate api (sr25519 signed tx), 图中的路径1
- [x] `AccountId_substrate ->  AccountId_ethereum`:  substrate api (sr25519 signed tx), 图中的路径1
- [x] `AccountId_ethereum -> AccountId_ethereum`:  eth api (ecdsa signed tx), 图中的路径2
- [ ] `AccountId_ethereum -> AccountId_substrate`: substrate提供ecdsa账户和ethereum账户并不兼容, 虽然frontier提供了图中路径3的方式(pallet-evm withdraw), 但并不通用.需要自定义`Ecdsa Signature`兼容ethereum账户, 比如Moonbeam的解决方案.图中路径`3`,`4`,`5`都是一种可能的技术路径.

本文介绍的是路径`5`的一种实现: 通过eth api调用evm contract完成ethereum的验签, 在contract内部再调用precompile contract, 而precompile的实现则是直接调用pallet-balances transfer.

## 3. Precompiled Contract

预编译合约是 EVM 中使用的一种方案，用于提供不适合在操作码中编写的更复杂的库函数（通常用于加密、散列等复杂操作）。它们适用于简单但经常调用的合约，或者逻辑固定但计算量大的合约。预编译合约在客户端使用客户端代码实现，并且由于它们不需要 EVM，因此运行速度很快。

在frontier的EVM实现中，通过预编译合约可以实现EVM调用WASM合约.

本文中, 我们是通过编写的solidity contract调用了pallet-balances的transfer函数.

下面是我们在SherpaX上运行的预编译合约的代码片段:

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

withdraw 预编译合约的合约地址是0x401(即1025),  input字节数组中包含了从evm contract中传入的参数
input = from(evm address, 20 bytes) + to(substrate pubkey, 32 bytes) + value(32 bytes)

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
## 4. Solidity Contract

ethereum账户向substrate 账户转账的合约, 
参数to是bytes32类型，是接收者的公钥, 参数value是转账的金额数量

```solidity
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

## 5. 演示结果

我们在SherpaX上的测试
- solidity contract address: `0xc01Ee7f10EA4aF4673cFff62710E1D7792aBa8f3`
- ethereum account: `0xf24FF3a9CF04c71Dbc94D0b566f7A27B94566cac`
- substrate account:  `Fr4NzY1udSFFLzb2R3qxVQkwz9cZraWkyfH4h3mVVk7BK7P`,  it's public key `0x90b5ab205c6974c9ea841be688864633dc9ca8a357843eeacf2314649965fe22`,  balance is 0.
- transfer balance: 1000000000000000000 wei =  1 KSX

expect balance after transfer: 1000000000000000000 wei =  1 KSX

![image](https://user-images.githubusercontent.com/8869892/131990239-49d17073-63fd-4c10-8427-8af2fae66f27.png)

![image](https://user-images.githubusercontent.com/8869892/131985687-986b73db-bb9f-41e1-adfa-72be89ac775e.png)

![image](https://user-images.githubusercontent.com/8869892/131985808-cdcfa7c3-3d7e-4b4d-bf72-b6c7adc078f8.png)

