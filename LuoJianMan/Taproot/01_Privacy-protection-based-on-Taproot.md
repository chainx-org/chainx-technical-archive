# Privacy protection based on Taproot

## Contenent

1. [Overview of Taproot](#Overview%20of%20Taproot)
2. [The current privacy issues of the Bitcoin network](#The%20current%20privacy%20issues%20of%20the%20Bitcoin%20network)
3. [Schnorr aggregated signatures protect multiple parties](#Schnorr%20aggregated%20signatures protect%20multiple%20parties)
	+ [As a signature algorithm for the Bitcoin network](#As%20a%20signature%20algorithm%20for%20the%20Bitcoin%20network)
	+ [Support aggregation signature to achieve privacy protection](#Support%20aggregation%20signature%20to%20achieve%20privacy%20protection)
4. [MAST provides more privacy for complex transactions](#MAST%20provides%20privacy%20protection%20for%20trading%20scripts)
5. [Users' privacy under Taproot upgrade](#Users'%20privacy%20under%20Taproot%20upgrade)

## Overview of Taproot

​		Privacy, extensibility, and security are the main issues of the day. Despite being the world's most popular cryptocurrency, Bitcoin still needs to address these issues, Taproot was born for this. Taproot consists of BIP340, BIP341, and BIP342, which contain Schnorr Signature, MAAST, and TapScript, respectively. The upgrade, the biggest to date for the Bitcoin network, is expected to profoundly change some of the current dilemmas facing the currency's development.

## The current privacy issues of the Bitcoin network

​		In Bitcoin networks, personal transaction records or merchant cash flow can be tracked by anyone. For further, to the currency, the types of multiple signature trading networks, for cooperation to complete a deal, but in the current bit network, when trading output perform unlocking condition, it will display the entire script and it contains all of the data, network participants can through the use of the early hash value of the script in the chain of blocks on the audit. These problems arise because the design of Bitcoin itself does not take into account this level of privacy protection, which is an issue that blockchain urgently needs to address.

## Schnorr aggregated signatures protect multiple parties

### As a signature algorithm for the Bitcoin network

​		ECDSA stands for Elliptic Curve Digital Signature Algorithm. Its role in Bitcoin is not strange to us. We use ECDSA technology every time we sign in to the Bitcoin network. For example, if User A wants to send A transaction to User B, the miner must confirm that only User A has the private key of the UTXO, which means that User A has the right to dispose of the asset. Therefore, User A needs to use ECDSA to generate A unique and unmodifiable digital signature to prove that it owns the private key and confirm the specific amount of the transaction part. Since the creation of Bitcoin, the algorithm has done a good job of generating signatures and maintaining the security of the Bitcoin network, but ECDSA lacks formal proof of security and relies on additional assumptions. Schnorr and ECDSA are also based on the discrete logarithm problem, but the Schnorr signature algorithm has formal mathematical proof to prove its security and uses fewer assumptions. Therefore, from the perspective of security, it is not surprising that the Schnorr signature algorithm takes over the ECDSA algorithm.

### Support aggregation signature to achieve privacy protection

![aggreviate](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626285815309-1626285815302.png)

​		In addition, the biggest advantage of Schnorr lies in the realization of aggregated signatures, because Schnorr has linear characteristics, which allows the public keys of multiple users to be aggregated into one public key through linear calculation, and the corresponding aggregated signature can be generated. Aggregation means merging into one, so when multiple parties participate, Schnorr aggregate signature can compress and merge multiple signature data into a single aggregate signature. The verifier signs a single aggregate through a list of all signature-related data and public keys. Perform verification. If the verification is passed, the effect is equivalent to independent verification of all relevant signatures and all of them are passed. 
​		Taking 2-3 multi-sign as an example, the current locking script of Bitcoin multi-sign requires 3 public key addresses, which will be compressed into a script, so the size will not change after upgrading. However, the unlocking script requires 2 public keys and 2 signatures. After upgrading to Schnorr, only one "public key and" and "signature and" is needed. For the more general N-M multi-signature, currently, the Bitcoin multi-signature unlocking script requires N public keys and N signatures, while Schnorr signature still requires only one "public key and" and one "signature and". In other words, the more signers, the higher the space utilization of the Schnorr signature.
​		Aggregating the signatures of multiple participants into a single signature makes a multi-signature transaction look like a regular P2PKH transaction, protecting the privacy of multi-signed participants. By contrast, ECDSA itself does not support multiple signatures. Bitcoin is now processed through a P2SH script, but a P2SH script exposes the existence of multiple signed transactions to the network and discloses all signers. Therefore, the use of the Schnorr signature, enhances the privacy of the transaction, and saves the space caused by multiple signatures in the unlocking script, saves precious on-chain space, realizes disguised capacity expansion.

## MAST provides privacy protection for trading scripts

![MAST](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626289416148-1626289416142.png)

​		The MAST(Merkle Abstract Syntax Tree), developed from both Abstract Syntax trees and Merkle trees, the technique behind the AST allows us to split a script into mutually exclusive subsets. Merkle trees allow us to verify that individual subsets of scripts belong to a complete script without disclosing the entire script.
​		MAST uses Merkle trees to code mutually exclusive branches of scripts, which allows complex script conditions to increase privacy by hiding branch scripts that are not executed. Specifically, when a user makes a payment, he only needs to disclose the relevant script and the path from the script to the root of the Merck tree, and a Merck proof can provide proof for the executed script.

![op_code](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626330184926-1626330184925.png)
		

​		Taking Alice's property-handling script above as an example, the script specified above includes not only Alice's public key (which needs to verify the signature of the private key), but also Bob's and Charlie's public keys and some conditional logic, such as timeout. In the current BTC network, all the data and script costs mentioned above are recorded on the chain, so everyone can track all the information of the UTXO, which is bad news for Alice, Bob and Charlie's privacy.

​		With the introduction of MAST, we can see the MAST tree's representation of the script as follows.

![mast-eg](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626314836408-1626314836407.png)

​		Divide it into two subscripts. One script is that Alice can spend the bitcoin at any time, and the other script is that Bob and Charlie can decide what to do with the bitcoin when Alice's bitcoin is not used up in three months. In reality, only one branch is selected for execution, and by exposing a single branch, it is impossible to determine what Alice's other subscripts are doing, thus protecting Alice's privacy.

## Users' privacy under Taproot upgrade

​		Taproot itself is a soft fork upgrade. After the Bitcoin client is upgraded, it will be compatible with the old client. This is powerful for the development of Bitcoin itself and the cohesion of the community. Taproot enhances the privacy of Bitcoin network participants by hiding all the scripts of transactions while making complex transactions indistinguishable from other transactions. For ordinary users who do not use complex transaction scripts, this Taproot upgrade is almost imperceptible; but for developers, wallets and services must be upgraded with corresponding functions. As more and more users take advantage of Taproot's features, its positive impact on efficiency and privacy is magnified.

​		In short, compared to the current Bitcoin network, blockchain analysis can track a single address. Taproot has incorporated certain privacy functions into the Bitcoin network. The protection of user privacy will also benefit Bitcoin itself and promote The development of the Bitcoin network has brought more possibilities for the future of Bitcoin.
