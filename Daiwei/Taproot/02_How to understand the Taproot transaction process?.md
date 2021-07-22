# How to understand the Taproot transaction process?

The core of Taproot is composed of Schnorr signature and MAST abstract syntax tree, and the Taproot transaction process is explained in two aspects: the creation of Taproot public key and Taproot spending mode.



## Create Taproot public key

To create a Taproot public key, you first need to understand the generation process of the Taproot aggregate public key and aggregate signature. At present, relevant researches with a high degree of attention include MuSig1 and MuSig2. Compared with the two rounds of MuSig2 communication, the biggest disadvantage of MuSig1 is that it requires three rounds of communication to create a signature, and each round of communication consists of messages passed back and forth. Since this article does not involve communication between networks, the main discussion here is the basic generation process of aggregated public keys and aggregated signatures.

### Aggregate public key

![test](https://github.com/bitcoinops/taproot-workshop/blob/Colab/images/musig_intro_1.jpg?raw=1)

From the above figure, it can be seen that the process of generating aggregated public keys can be divided into three steps: 1) Transfer public keys to each other, and concatenate all public keys to perform an aggregate hash to generate c_all. 2) c_all and the public key of i are hashed together to generate factor c_i. 3) Perform linear combination according to the factor c_i to obtain the aggregate public key P_agg.

![image-20210722091644213](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722091644213.png?raw=true)

### Aggregate signature

![test](https://github.com/bitcoinops/taproot-workshop/blob/Colab/images/musig_intro_2.jpg?raw=1)

The above figure shows that three rounds of communication are required, which are mainly divided into two major steps: 1) Generate nonce and linearly aggregate to generate R_agg. 2) Each party uses the private key to generate the Schnorr signature and aggregate to obtain the final aggregate signature (R_agg, s1+s2+s3).

![image-20210722091724750](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722091724750.png?raw=true)

### Join MAST

![img](https://github.com/bitcoinops/taproot-workshop/blob/master/images/taptree0.jpg?raw=true)

As shown in the figure above, the Taproot public key is mainly composed of two parts, including the aggregate public key P and the public key tG formed by the MAST structure. Assume that P is the aggregate public key of Alice, Bob, and Charlie, and script_A, script_B, and script_C are scripts related to Alice, Bob, and Charlie. Then, the Taproot public key creation process is as follows:

1. Alice, Bob, and Charlie each create general public and private keys.

   ![image-20210722092748689](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092748689.png?raw=true)

2. The public key is aggregated into pubkey_agg, and the private key is adjusted for future signatures.

   ![image-20210722092816042](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092816042.png?raw=true)

3. Create script script_A, script_B, script_C

   ![image-20210722092854675](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092854675.png?raw=true)

4. Construct the MAST abstract syntax tree and calculate the private key taptweak corresponding to the MAST structure. In the above figure, TaggedHash represents a hash with a tag, a fixed length of 32 bytes, and the calculation method is TaggedHash(tag, x) = sha256(sha256(tag) + sha256(tag) + x); ver represents the Tapscript version number, and the current value is 0xc0; size indicates the number of bytes of script; A&B means that A and B are sorted by dictionary and then byte spliced.

   ![image-20210722092920158](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092920158.png?raw=true)

4. According to the formula Q=P+tG, the taproot public key is synthesized, and segwit_address is generated for the following transactions.

   ![image-20210722092943482](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092943482.png?raw=true)

5. Transfer 50 BTC to Taproot segwit address

   ![image-20210722093015597](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093015597.png?raw=true)

## Taproot cost

To complete the transfer of 0.5 BTC from the above Taproot address to the Bob individual, there are two payment methods: one is Alice, Bob, and Charlie all sign and forms an aggregate signature to complete the transfer to Bob; the other is through the MAST structure The script transfers money to Bob.

1. Create the original transaction, fill in the recipient's address, transfer amount, and other data.

   ![image-20210722093042214](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093042214.png?raw=true)

2. According to the first method to transfer money, first Alice, Bob, and Charlie need to generate nonces and aggregate them, and then each uses a private key to perform Schnorr signatures and finally perform signature aggregation operations. Therefore, the final witness script in this way is a single signature with a fixed length of 64 bytes.

   ![image-20210722093114659](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093114659.png?raw=true)

3. Test the legality of the original transaction of the first type of cost structure and send the transaction

   ![image-20210722093202683](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093202683.png?raw=true)

4. According to the second method, if Alice completes the transfer to Bob through script_A, then the witness script needs to include: 1) `[Stack element(s) satisfying TapScript_A]` 2)`[TapScript_A]` 3)`[Controlblock c]`. Among them, `[Controlblock c]` represents the proof related to `TapScript_A`, the length is 33+32n. The first byte of the 33 bytes is calculated by the aggregate public key and the Taproot version number, and the remaining 32 bytes represent the x coordinate of the aggregate public key. 32n represents the proof of `TapScript_A`. In this example, n=2, which refers to `taggedhash_leafB` and `taggedhash_leafC`.

   ![image-20210722093350335](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093350335.png?raw=true)

5. Test the original text of the transaction formed by the second type of cost is legal and send the transaction

   ![image-20210722093409770](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093409770.png?raw=true)

## Conclusion

In general, Taproot transactions are mainly concerned with one output and two spending mode. One output makes the public key in the lock script consistent regardless of whether it is a personal transaction or a multi-signature transaction, and it cannot be distinguished from the form. The two spending modes allow us to use fewer bytes to achieve a more complex transaction process.