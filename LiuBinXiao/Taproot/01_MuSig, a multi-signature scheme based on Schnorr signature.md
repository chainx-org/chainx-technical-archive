## Overview

​		As the Taproot upgrade gets closer and closer, many people have a keen interest in the MuSig multi-signature scheme. MuSig allows a group to collectively own some bitcoins and create a single signature to authorize payment. Due to the innovative key aggregation feature of MuSig, this signature is a regular Schnorr signature. Once Taproot is activated, it can be processed by Bitcoin. When used to create multiple signature wallets, MuSig reduces transaction costs and increases privacy compared to traditional n signatures using CHECKMULTISIG opcodes, which require n public keys and n ECDSA signatures on the blockchain.

## Multi-signature scheme

​		The multi-signature scheme is a combination of signature and verification algorithms, in which multiple signers (each person has their private key/public key) jointly sign a message to produce a single signature. Then, anyone who knows the message and the signer's public key can verify this single signature. Note that in the context of Bitcoin, the term "multisig" usually refers to a k-of-n strategy, where k may be different from n. In cryptography, multi-signature only involves n-to-n strategies, although we can easily construct k-to-n in the case of n-to-n.

## MuSig

​		MuSig is a multi-signature scheme that uses the Schnorr digital signature algorithm to aggregate public keys and signatures. MuSig allows multiple users to use their private keys to create a combined public key, which is the same size and indistinguishable from any other Schnorr public key, including the public key of a single user. It further describes how users who create the public key can jointly and securely create a signature corresponding to the public key. As the public key, this signature is no different from other Schnorr signatures. Compared with traditional script-based multisig, MuSig uses less block space and has more private space, but it may also require more interactivity between participants.

### MuSig supports key aggregation

​		MuSig supports key aggregation. Key aggregation refers to a multi-signature that looks like a single-key signature but is opposed to an aggregated public key that only functions as the public key of the participant. Therefore, the verifier does not need to know the public key of the original participant, just gives them an aggregated key. MuSig can generate short, fixed-volume signatures, no matter how many signers there are and how they sign, they will look the same to the verifier. In a blockchain system, verification efficiency is the most important factor. Unless more security is needed, there is no need to provide the verifier with more details about the signer. This has an obvious advantage in that it can improve privacy because it can hide the information of the specific signer. Therefore, MuSig is a key aggregation scheme for Schnorr signatures.

### MuSig has provable security in the common public key model

​		MuSig has provable security in the common public-key model. Many multi-signature schemes provide key aggregation for Schnorr signatures, but they have some limitations, such as the need to verify that participants have a private key corresponding to the public key they claim to own. It can be proved that there are no restrictions on the security in the common public key model, and participants only need to provide their public keys. This means that the signer can use ordinary key pairing to participate in multi-signature, without any need to provide any information about the specific methods of production and control of these keys. In some Bitcoin scenarios, the key management policies of individual signers are different and restricted. At this time, it is difficult to obtain information about key generation, which greatly enhances the privacy of users.

### MuSig interaction problem

​		While realizing the above advantages, it also brings some problems. One of the more obvious problems is that MuSig requires more interaction between signers. To be more precise, creating a signature requires three rounds of communication, and each round of communication consists of messages passed back and forth. The following figure shows the process of three rounds of communication. You can imagine one signer is a desktop wallet and the other is a Blockstream Green co-signer, or the signer shares a Lightning channel that they are trying to close.

![MuSig rounds](https://miro.medium.com/max/553/0*f1uUkZ0BCB1aQrC0.png)

​		In contrast, wallets using CHECKMULTISIG only need one round of communication: they receive a transaction and return a signature. For example, if you use MuSig to forward payments in the Lightning Network, privacy will be improved, but the payment time will be significantly longer. As the communication delay increases, this problem will become more serious. The MuSig signature device stored in the safe requires its owner to visit twice before creating a signature.

## MuSig2

​		As the name suggests, MuSig2 is the successor of MuSig. It provides the same functionality and security as MuSig, but can eliminate almost all interactions between signers. With MuSig2, the signer only needs two rounds of communication to create a signature, and the key is that one of the rounds can be preprocessed before the signer knows the message they want to sign. Once some messages need to be signed, for example, a Bitcoin transaction, the process is the same as the current checkmultisigns-based wallet: the transaction is transmitted to the signer, and then a signed reply is received. MuSig2 retains the simplicity and efficiency of MuSig, but adds a small number of additional calculations.

### Concurrent session security

​		MuSig2 is safe under concurrent sessions. In MuSig, each signer creates a nonce, in MuSig2, each signer creates two or more nonces `R_i,1, R_i,2` in the first round and send them to other signers, and they are valid Use a random linear combination `R_i = R_i,1 + b*R_i,2` instead of the previous nonce `R_i`. The coefficient b is the output of the hash function applied to all signers' non-parameters, aggregate public keys, and messages. Like MuSig, the aggregation is `R = R_1 + ... + R_n`. If any signer changes any of their non-messages, every other signer will use a different, random linear combination of their two non-messages. This avoids the attacks found in the other two rounds of multi-signature schemes.

## Summary

​		With MuSig2, some protocols will benefit greatly, such as "[scriptless script Lightning](https://github.com/ElementsProject/scriptless-scripts/blob/master/md/multi-hop-locks.md)" and threshold signature. If Taproot is started, MuSig2 can be applied to a series of Blockstream products, such as Blockstream Green and c-lightning, as well as the Liquid peg mechanism. The Liquid network may activate Taproot before Bitcoin, and we can experience MuSig2 on it earlier!