# Cross-Chain Technology

# Why cross-chain?

Speaking of DeFi, which has the most users at present, according to the statistics of DeFi Llama, the lock-up volume of DeFi on Ethereum has exceeded 100 billion U.S. dollars. Other public chains such as BSC, Solana, Avalanche have also attracted 49 billion U.S. dollars of funds. 

Although the funds of many public chains are already quite large, different chains are like islands, and assets on different chains cannot be exchanged freely. In addition, many emerging public chains still lack a lot of infrastructure, such as stable coins, which are neither as strong as Tether or Circle. The original stable currency collateralized by fiat currencies issued by powerful centralized institutions, and there is no decentralized stable currency issued by over-collateralization with cryptocurrencies such as ETH with relatively small price fluctuations as collateral.

Therefore, it is necessary to introduce assets on other chains into its own public chain through cross-chain. Among the currently commonly used cross-chain methods, in addition to cross-chain withdrawals in centralized institutions such as exchange wallets, the most common are various Centralized cross-chain asset bridge.

## Cross-Chain Bridge

A cross-chain bridge is a connection method that transfers tokens or data between blockchains. The two chains can have different protocols, rules and governance models. The cross-chain bridge provides a compatible way to secure between the two. To interoperate.



How do two independent blockchains know what is happening on the other chain? This is actually an oracle problem. The simplest solution currently is to allow multiple nodes to monitor contract events on the blockchain at the same time. When the vast majority of nodes agree that they have seen the event, it can be considered that the nodes have reached a consensus and trigger the next step in the sequence. An event.

## Multi Chain Currency

To use assets on one chain on another chain, you must have the same asset on both chains to form a multi-chain token. When new assets are generated on the target chain, the assets on the old chain can be directly destroyed or pledged in a specific contract.



## Cross-chain bridge model

According to the way of reaching a consensus and whether escrow is needed, cross-chain bridges can be divided into the following categories.

- Hosting+ centralization (ChainX bridge v1, such as centralized exchange cross-chain, WBTC, etc.)
- Hosting + Over-collateralization (DAI, PolkaBTC, ChainX bridge v2)
- Hosting+ PoS (Matic, xDAI)
- Hosting + MPC (multi-party computing) (Thorchain, Anyswap)
- Non-custodial + MPC (Multichain)

The cross-chain bridge of a centralized exchange is the most convenient for users to use, but at the same time there may be a single point of failure. Most cross-chain bridges are hosting user assets. How to reach a consensus is also very important for cross-chain bridges, which is related to the security of assets under custody. The current cross-chain is also gradually developing in the direction of non-custodial.

## ChainX bridge v2

Currently ChainX has implemented two cross-chain technologies, one is the semi-centralized 1.0 solution based on multi-signature, and the other is the 2.0 solution based on over-collateralization that will be launched soon. In the 2.0 solution, the role of Vault (Vault) is introduced. Vault obtains the eligibility to keep BTC through over-collateralization of pcx. The safe collateral rate is 300%. Remember that the maximum value of BTC that Vault can hold is $V_max$, Vault’s current collateral is $C$, and the exchange ratio between BTC and PCX is $E$, then there is:
$$
V_{max} = \frac{C}{E}
$$
The over-collateralization scheme is a decentralized cross-chain scheme. It is very flexible for other schemes. Any user can become a Vault by staking pcx. From a security perspective, cross-chain assets are scattered among many users, and the actions of individual users will not affect the overall operation. Under the "black swan" incident, users can also recover their asset losses through forced redemption.

## Summary

In the era of DeFi, the original model of independent chain is no longer sufficient to adapt to the market. Value interoperability is the future of the market. ChainX, as the Layer 2 of BTC, paves the way for the derivative applications of BTC, and these infrastructures are the security and stability of the cross-chain solution, and ChainX will also work on this.