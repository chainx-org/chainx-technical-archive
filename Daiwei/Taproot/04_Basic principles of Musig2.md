# Basic principles of Musig2

## Musig2 introduction

​	The multi-signature scheme enables a group of signers to generate a joint signature on a message. Musig2 is a multi-signature scheme and a new variant of the MuSig signature scheme. MuSig allows multiple signers to create an aggregate public key from their respective private keys, and then jointly create a valid signature for the public key. The aggregate public key created in this way is indistinguishable from other public keys. The original MuSig requires three rounds of signatures, but the new MuSig2 scheme implements a simple signature protocol that only requires two rounds and does not require zero-knowledge proofs. MuSig2 is a simple and highly practical two-round multi-signature scheme with advantages: i) it is secure under concurrent signing sessions, ii) supports key aggregation, iii) outputs ordinary Schnorr signatures, iv) only requires two rounds of communication , V) has signer complexity similar to ordinary Schnorr signatures. At the same time, Taproot is upgraded to be compatible with the Musig2 multi-signature scheme. In our new ChainX cross-chain scheme, the Musig2 signature scheme will be used to complete the process of aggregating signatures.

## Musig2 principle

![Musig2_1](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_1.png?raw=1)

​	The above picture is from the original paper and describes the basic process of Musig2 signature. Musig2 signature first requires the participants to generate public and private keys through `KeyGen()` and temporary public and private keys (i.e. nonce) through `Sign()`. Then the first round of communication is carried out, and the temporary public key (i.e. `out_i` in the figure above) is passed to others. After getting the temporary public keys of other parties, the signature fragment `out'_i` can be generated through the two functions `SignAgg()` and `Sign'()`. Finally, the second round of communication is carried out, and the signature fragments `out'_i` are passed to others. After getting the signature fragments of other parties, the final result can be generated by `SignAgg'()` and `Sign''()` Sign `σ`. In the figure, `KeyAgg()` represents the public key aggregation function, and `Ver` represents the Musig2 signature verification function. The final form of Musig2 aggregate signature is the same as that of Schnorr's ordinary signature, which can be expressed as (R, s). The following shows part of the implementation of the rust version of Musig2.

### Sign

1. Generate key: Each party generates a random private key and calculates the corresponding public key

   ![Musig2_2](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_2.png?raw=1)

2. Generate nonce: All parties generate temporary public and private keys. In the function `create_vec_from_private_key()`, Musig2 will generate a temporary public and private key of `Nv` for two rounds of communication to complete the aggregation signature. The return value of the `sign()` function `msg` contains all the temporary public keys, which can be passed to others through the p2p network. In the code below, the process is simulated by `party1_received_msg_round_1` and `party2_received_msg_round_1`.

   ![Musig2_3](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_3.png?raw=1)

3. Generate signature fragments: The following code uses the `sign_prime()` function to complete the functions of the two functions `SignAgg()` and `Sign'()` in the paper. Among the parameters, `message` represents the message to be signed, `pks` represents an ordered list of all public keys of each participant, and `party_index` represents the index of the public key of the current participant in `pks`. The `sign_prime()` function mainly uses the `c, R, b_coefficients` aggregated by the private key `self.keypair` and the temporary public key `msg_vec` to generate the signature fragment `s_i`. Here `c` represents the message digest generated by hash generation by `message`, `R` is the first item of the final result of Musig2 aggregated signature, and `b_coefficients` is the coefficient for generating signature fragments. Pass the signature fragment `s_i` to others through the p2p network. In the code below, the process is simulated by `party1_received_msg_round_2` and `party2_received_msg_round_2`.

   ![Musig2_4](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_4.png?raw=1)

4. Aggregate signature fragments: After collecting signature fragments from other parties, generate the second item of the Musig2 aggregation signature through `sign_double_prime()`. This function mainly accumulates signature fragments. `s_total_1` and `s_total_2` represent the aggregate signature fragments generated by the participating parties. Normally, the two are the same.

   ![Musig2_5](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_5.png?raw=1)

### Verification

![Musig2_6](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_6.png?raw=1)

​	The above figure shows the verification process of Musig2 through the `verify()` function. In the parameter `r_x, signature` corresponds to the (R, s) of the Musig2 aggregated signature, `X_tilde` represents the aggregated public key, and `c` is consistent with the previous text, indicating the message digest generated by the hash generated by `message`.

## Conclusion

​	In general, Musig2 only needs to pass two rounds of communication to complete the process of aggregating signatures. Compared to Musig, the main difference lies in the nonce generation process. Among them, each participant of Musig2 generates multiple pairs of temporary public and private keys, while Musig only generates a pair of temporary public and private keys. This is also the fundamental reason why Musig2 only needs two rounds of communication. At the same time, Musig2 is a multi-signature scheme based on the basic principles of Schnoor signatures, which can be well compatible with Taproot upgrades and applied to our new ChainX cross-chain upgrade scheme.