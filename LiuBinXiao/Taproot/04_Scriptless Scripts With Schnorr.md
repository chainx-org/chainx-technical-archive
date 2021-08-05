## Content Guide

- [Scriptless Scripts](#Scriptless-Scripts)
- [Schnorr Signatures](#Schnorr-Signatures)
- [Adaptor Signatures](#Adaptor-Signatures)
- [Atomic (Cross-chain Swaps) Example with Adaptor Signatures](#Atomic-(Cross-chain-Swaps)-Example-with-Adaptor-Signatures)
- [Summary](#Summary)

## Scriptless Scripts

Scriptless Scripts was created by mathematician Andrew Poelstra to further explore the mathematical characteristics of Schnorr signatures. It is a method of executing smart contracts off-chain using Schnorr signatures. At present, smart contracts require the entire network to obtain and execute code logic. Although it is very simple for everyone to execute scripts, it is not necessary to execute by the blockchain, and the script logic itself is public, which is fundamentally not private enough. Executing smart contract logic off-chain is the goal of the Scriptless Scripts approach. To implement smart contracts off-chain, the spending conditions are not executed by the blockchain, but by the parties themselves. Only when the parties to the contract agree and meet the conditions, they will sign and execute the final transaction. For the blockchain, it looks like an ordinary signature, and only the participants know what happened.

## Schnorr Signatures

With Schnorr signatures, you can have a public key, but it is the sum of the public keys of many different people. The resulting public key is a public key that can verify the signature-only all participants can cooperate to generate the signature, so it is still a multi-signature. Multi-signature (multisig) has multiple participants who generate signatures. Each participant may generate a separate signature and connect them to form a multi-signature. Of course, here we need to look at the basics of Schnorr signatures first. First, the signer has a private key x and a random private nonce r. G is the generator of the discrete logarithm group, R=rG is the public random number, and P=xG is the public key. Then you can simply calculate the signature s of the message m linearly as follows:

![schnorr](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960316485-1627960316469.png)

The verification equation will multiply each term in the equation by G and consider the cryptographic assumption (discrete logarithm), that is, G can use multiplication but cannot use division, thus preventing deciphering.

![verify](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960381309-1627960381308.png)

The elliptic curve digital signature algorithm (ECDSA) signature is not linear in x and r, so it has little effect. But for Schnorr signatures, all s will be aggregated into multi-signatures, just a simple calculation like the following:

![sum](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960439005-1627960439004.png)

It can be clearly seen that these signatures only need to be calculated off-chain and are essentially Scriptless Scripts. In this way, the independent public keys of multiple participants will be aggregated to form a single key and signature, and the detailed information of the number of participants and the original public key will not be revealed when publishing.

## Adaptor Signatures

Schnorr's multi-signature can be modified to generate an adapter signature, which will serve as the basic part of all Scriptless Scripts functions.

The adapter signature will "hide a secret" in the signature so that when the payee asks for funds, they must reveal the secret to the payer. Suppose t is the payment secret (known by the receiver), and T=t*G is the point/public key related to this secret (known by both parties). As with normal payments, the payer/sender of the funds will generate a Schnorr signature that sends the funds transaction to the receiver. Different from normal payment, the sender will use T to adjust their signature, while the receiver can only repair this signature by using the secret t to obtain a valid signature. This generated invalid signature will be sent to the recipient (not the blockchain), and the recipient will use their secret to repair the signature before broadcasting the transaction to the blockchain using the now valid signature. Therefore, the adapter signature is not a complete and valid signature, but a promised signature that will reveal the secret. This concept is similar to the concept of atomic swap, except that no script is implemented.

Here, the Schnorr multi-signature structure is modified so that the first party generates:

![gen](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960536172-1627960536172.png)

Where t is the shared secret, G is the generator of the discrete logarithm group, and r is a random number.
Using this information, the second party generates:

![gen2](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960571916-1627960571916.png)

The coin to be exchanged is contained in the message m. The first party can now calculate the complete signature s such that

![s](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960611803-1627960611802.png)

Then the first party calculates and publishes the adapter signature s' to the second party (and anyone else)

![s'](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960646893-1627960646892.png)

The second party can use s'G to verify the adapter signature s'

![s'G](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960688370-1627960688369.png)

However, this is not a valid signature because the random number point of the hash is R+T instead of R. The second party cannot retrieve a valid signature from it, and needs an ECDLP solution to recover s'+t, but this is actually impossible. After the first party broadcasts s to receive the coins in the message m, the second party can calculate the key t through the following formula:

![t](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627960725200-1627960725192.png)

## Atomic (Cross-chain Swaps) Example with Adaptor Signatures

A simple but powerful application of adapter signatures is to create a cross-chain atomic exchange, which allows trustless exchange of assets. The adapter signature is a signature that can be offset by a value. Once combined with the real signature, the receiver can calculate the sender's key t. The adapter signature can be verified as authentic, but no information will be disclosed until the real signature is published. These adapter signatures enable us to achieve atomicity in cross-chain atomic swaps. Holders of the adaptor's signature can rest assured that if their opponent claims their coin, they will be able to claim their coin.

Here is a simple example. Suppose Alice has a certain number of coins on a specific blockchain; Bob also has a certain number of coins on another blockchain. Alice and Bob want to exchange atoms. However, the two blockchains do not know each other and cannot verify each other's transactions. The classic way to achieve this goal involves using the blockchain's scripting system to conduct a hash pre-image inquiry, and then display the same pre-image on both sides. Once Alice knew the original image, she would take it out to take her coin. Then Bob copies it from one chain to the other to get his coins. Using adapter signatures, the same result can be achieved in a simpler way. In this case, both Alice and Bob put their coins on two of the two outputs of each blockchain. They sign the multi-signature agreement in parallel, and then Bob uses the same value T to provide Alice with each party's adapter signature. This means that if Bob wants to take his coin, he needs to reveal t; in order for Alice to take her coin, she needs to reveal T. Then Bob replaces one of the signatures and issues t, taking his coins. Alice calculates t based on the final signature visible on the blockchain and uses it to display another signature, giving Alice her coin.

It can be seen that the adapter signature achieves atomicity. There is no clear hash or pre-image on the blockchain, but people can still exchange information, and privacy is achieved without the need for scripting attributes.

## Summary

Scriptless Scripts are more scalable than standard smart contracts because the execution of the contract takes place off-chain. By pushing this execution to those who care about it, there is no need to consume public computing resources to store contract data and execute the contract. In terms of privacy, moving the logic and execution of smart contracts from on-chain to off-chain will increase privacy. When on the chain, a lot of information about the smart contract will be shared with the entire network, including the number and addresses of participants, and the amount of money transferred. By moving the smart contract off-chain, the network only knows that the participants have agreed to the contract and the related transaction is valid. Only the parties know the details of the contract, and the visible things are no different from other ordinary transactions. This provides us with a feature called repudiation, which means that for a neutral observer, transactions are indistinguishable from ordinary transactions. In terms of efficiency, Scriptless Scripts minimizes the amount of data that needs to be verified and stored on the chain. By moving the smart contract to the chain, the cost of the full node is less, and the user's transaction fee is lower.

