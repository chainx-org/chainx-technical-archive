# Pay-to-Taproot (P2TR)

## P2TR Introduce

​		Pay-to-Taproot (P2TR) is a ScriptPubKey that locks bitcoins to a script, allowing users to pay to the Merkle root of Schnorr public keys or various other scripts. On the surface, a P2TR output locks Bitcoin on a Schnorr public key, which we assume is Q. However, this public key Q is the sum of a public key P and a public key M. M is calculated by the Merkle root of other ScriptPubKeys lists. The bitcoins in the P2TR output can be spent by issuing the signature of the public key P or satisfying one of the scripts contained in the Merkle tree. The former is called the key path and the latter is the script path. Although there are many ways to output P2TR, only the one that is used will be made public, so that privacy can be maintained for other unused alternatives. In addition, due to Schnorr key aggregation, the public key P itself can be an aggregation key, and the state of the public key P as an aggregation key or a single key will never be revealed, so all P2TR outputs are similar to each other. This will destroy many chain analysis heuristics and enhance user privacy.

## Other Payment Methods

### P2PKH

​		Pay-to-Public-Key-Hash (P2PKH) is a kind of ScriptPubKey, which locks Bitcoin to a public key hash (Bitcoin address). If Alice wants to send 1 BTC to Bob in a P2PKH transaction, Bob provides Alice with an address in his wallet. Bob's address is included in the transaction. When Bob tries to spend the bitcoin he received, he must sign the transaction with the private key corresponding to the public key, and the hash value of the public key is consistent with the hash value provided in Alice's transaction.

### P2WPKH

​		Pay-to-Witness-Public-Key-Hash (P2WPKH) is a ScriptPubKey used to lock Bitcoin to a SegWit address. The P2WPKH transaction is similar to the P2PKH transaction in most respects; it still locks Bitcoin to the hash value of the public key. The main difference is that P2WPKH uses SegWit. This means that all input ScriptSig (the script to unlock Bitcoin) is moved out of the transaction body and enters the witness part, which is called the script witness. These data are still recorded on the blockchain, but the cost of data will be lower than regular data, making SegWit transactions cheaper than regular transactions.

### P2SH

​		Pay-to-Script-Hash (P2SH) is a kind of ScriptPubKey, which is mainly used for multi-signature wallets to make output script logic and check multi-signature before accepting transactions. For example, if Alice sends 1 BTC to Bob in a P2SH transaction, she will include the hash of the script required to spend Bitcoin in the transaction. This script may require Bob's private key and/or the signature of many others. When Bob wants to spend the bitcoin he received from Alice, he reconstructs the hash of the script that Alice used to send the bitcoin and signs the transaction with any private keys required by the script. P2SH is very flexible because it allows users to build arbitrary scripts. In addition, the sender of the transaction does not need to know what script type they are sending to. In the above example, Bob can build the script he wants offline, and only send the hash value of the script to Alice, thereby preserving more privacy for Bob.

### P2WSH

​		Pay-to-Witness-Script-Hash (P2WSH) is a transaction type similar to P2SH transactions in most respects, except that it uses SegWit. Like P2SH transactions, P2WSH transactions lock Bitcoin to the hash of the script. To spend this bitcoin, the spender must present a script called RedeemScript and any necessary signatures. On a technical level, P2WSH describes the ScriptPubKey used to lock Bitcoin to the SegWit script hash.

## P2TR Advantages

​		By comparing the size of different types of signatures, it can be seen that the use of P2TR on a single signature is a bit larger than the equivalent P2WPKH, but closer observation will reveal that there are many uses of P2TR for single-signature wallet users and the entire network. benefit:

- Cost is cheaper: At the investment level, the cost of a single-signature P2TR UTXO is about 15% less than the cost of a P2WPKH UTXO. An over-simplistic analysis like the above table hides a detail, that is, consumers cannot choose the address they are required to pay, so if you stay on P2WPKH and everyone else upgrades to P2TR, your 2 in 2 out transaction is The typical size will be 232.5vbytes, while all P2TR transactions are still only 211.5vbytes.
- Privacy: Although early adopters will lose some privacy when they switch to the new script format, users who switch to taproot will also get an immediate increase in privacy. Your transactions will be able to look no different from those who are engaged in new LN channels, more effective DLCs, secure multi-signatures, various clever wallet backup, and recovery solutions, or a hundred other pioneering developments. Using P2TR for single signature now also allows your wallet to be upgraded to multi-signature, tapscripts, LN support, or other functions in the future without affecting the privacy of your existing users. It doesn't matter whether the old version or the new version of the software receives UTXO-both UTXOs look the same on the chain.
- More convenient for hardware signing devices: since the rediscovery of the fee overpayment attack, some hardware signing devices have refused to sign transactions unless each UTXO spent in the transaction has metadata, which contains the importance of the entire transaction that generated the UTXO Part of the copy. Taproot eliminates the potential vulnerabilities of fee overpayment attacks, so it can significantly improve the performance of hardware signers.
- More predictability: The ECDSA signature of P2PKH and P2WPKH UTXO can have different sizes. Since the wallet needs to choose the transaction rate before creating the signature, most wallets just assume the worst-case signature size, so accept smaller Will pay slightly more when signing. For P2TR, the size of the signature is known in advance, allowing the wallet to choose a precise fee rate.
- Help complete nodes: The overall security of the Bitcoin system depends on most Bitcoin users using their nodes to verify each confirmed transaction, including verification of transactions created by your wallet. Taproot's Schnorr signature can be effectively verified in batches. In the process of synchronizing blocks, the CPU cycles that nodes need to consume when verifying signatures are reduced by about 1/2. Even if you reject all of the above advantages, consider using taproot to help people running full nodes.

## Prepare For P2TR

​		For wallets that already support receiving and spending v0 segwit P2WPKH output, it should be easy to upgrade to v1 segwit P2TR for a single signature. Here are the main steps:

- Use the new BIP32 key derivation path: It is strongly recommended to use a new derivation path for the P2TR public key (for example, defined by BIP86). If you use the same key in ECDSA and schnorr signatures, you may be attacked.
- Adjust your public key by hash value: Although technically no single signature is required, especially when all your keys come from a randomly selected BIP32 seed, BIP341 recommends submitting your key to a non-consumable scripthash tree. This is as simple as using elliptic curve addition, adding your public key to the curve point of the key's hash value. The advantage of following this recommendation is that if you add support for scriptless multi-signature in the future, or add support for tr() descriptors, you will be able to use the same code.
- Create your address and monitor it: use bech32m to create your address. The payment will be sent to scriptPubKey OP_1 <tweaked_pubkey>. You can use any method used to scan v0 Segregated Witness addresses (such as P2WPKH) to scan transactions for payment scripts.
- Create a spending transaction: All non-witness fields of taproot are the same as those of P2WPKH, so you don't need to worry about changes in transaction serialization.
- Create a signature message: This is a commitment to the data of the spending transaction. Most of the data is the same as the data you signed for the P2WPKH exchange, but the order of the fields has been changed, and some additional things have been signed. Achieving this is just a matter of serializing and hashing various data, so writing code should be easy.
- Sign the hash value of the signature information: There are many different ways to create a Schnorr signature. So the best way at the moment is not to "launch your method", but to use features from a heavily vetted library you trust. However, if you can't do this for some reason, BIP340 provides an algorithm. If you already have the basis for making ECDSA signatures, then this algorithm should be easy to implement. When you have your signature, put it in the input witness data, and then send your spending transaction.

## Summary

​		The Pay-to-Taproot (P2TR) output is the SegWit output of version 1. All future Taproot transactions are SegWit transactions, so developers must give P2TR a try first. Before taproot is activated in block 709,632, you can use testnet, public default flags, or Bitcoin Core's private regtest mode to test your code.