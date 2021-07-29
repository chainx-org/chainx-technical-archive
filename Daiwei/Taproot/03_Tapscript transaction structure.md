# Tapscript Transaction Structure

## Tapscript

The Taproot upgrade is a compilation of three BIPs, which are Schnorr signatures (BIP 340), Taproot (BIP 341), and TapScript (BIP 342). Tapscript is the third part of the Taproot upgrade, supplementing Schnorr and Taproot. Tapscript mainly improves the upgrade mechanism and operation code.

Tapscript can easily achieve unforeseen future upgrades by using the opcode OP_SUCCESS. The opcode is an operator of the Bitcoin scripting language. Tapscript retains most of the opcodes applicable to the v0 witness script and updates a few opcodes.The changes in the Tapscript transaction structure mainly come from the lock script and the witness script. This article will introduce the composition of each field of the Tapscript transaction through a specific example.

## Opcode

The opcode changes in Tapscript include: 1) The `OP_CHECKSIG/OP_CHECKSIGVERIFY` opcode is changed from the original ECDSA signature verification to the verification Schnorr signature; 2) The two multi-signature opcodes `OP_CHECKMULTISIG/ OP_CHECKMULTISIGVERIFY` are disabled; 3) the opcode is added `OP_CHECKSIGADD`, replacing the original multi-signature opcode.

Since signature verification is the most CPU-intensive operation in Bitcoin scripts, the above-mentioned opcode changes are essential to achieve the efficiency improvements related to the Schnorr-based multi-signature scheme. The change of `OP_CHECKSIG/OP_CHECKSIGVERIFY` is naturally for the Schnorr signature to be used in the Taproot upgrade. The multi-signature opcode of `OP_CHECKSIGADD` requires the witness to provide a valid or invalid signature for each public key, thereby avoiding wasting the signature verification operation of each public key in the k-of-n multi-signature script.

The traditional 2-of-3 multi-signature transaction script is as follows:

- script: `2 [PK0] [PK1] [PK2] 3 OP_CHECKMULTISIG`
- Possible witness script: `[sig2]` `[sig1]`

Use Tapscript to implement the same multi-signature strategy, where the script is:

- Tap script: `[PK0] [OP_CHECKSIG] [PK1] [OP_CHECKSIGADD] [PK2] [OP_CHECKSIGADD] [2] [OP_NUMEQUAL]`
- Possible witness script: `[sig2]` `[sig1]` `[]`

It can be seen from the above script that the mapping relationship between the traditional multi-signature `sig` and `PK` is unknown. For the signature `sig1`, it is necessary to perform poll verification with `PK0`, `PK1`, and `PK2` respectively to determine whether `sig1` is verified or not. The same is true for `sig2`. Tapscript provides a valid or invalid signature for each public key. When the signature is valid, it is accumulated through the OP_CHECKSIGADD operation code. Finally, it only needs to verify whether the number of valid signatures reaches the threshold to determine whether the signature is passed. OP_CHECKSIGADD omits the polling process in traditional multi-signature verification and speeds up multi-signature verification.

## Transaction Structure

Tapscript follows the transaction structure in Segregated Witness and consists of the following fields. Tapscript stipulates that transaction input must be segregated witness expenses. This means that the script_sig in txins is generally empty, and there is data in the witness field. This article uses a specific example to illustrate the connotation of each field.

![Tapscript1](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%841.png?raw=true)

1. The following is the original text of a transaction from a Taproot address to an ordinary address. The original transaction is expressed in hexadecimal, and every two characters represent one byte. The Taproot address is composed of the aggregate public key and the public key corresponding to the MAST structure: Q=P+tG. Here P represents the aggregate public key of A, B, and C, and the calculation formula is `P=pubkey_agg(PKA, PKB, PKC)`. tG represents a separate script `[PKA] [OP_CHECKSIG] [PKB] [OP_CHECKSIGADD] [PKC] [OP_CHECKSIGADD] [2] [OP_NUMEQUAL]` as the public key calculated by the root of MAST, the calculation formula is `t=tagged_hash(P| |root)`.

   ![Tapscript2](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%842.png?raw=true)

2. As shown in the figure below, the first line indicates the field name, the number in `()` indicates the number of bytes of the corresponding field, and the second line indicates the corresponding data of the field in the original transaction text. nVersion represents the version identifier of the transaction structure. `02000000` is the little-endian mode, with the low byte first, so the actual value is `00000002`. Marker is a field whose value must be `00`, and the flag indicates whether there is a witness script, that is, whether there is data in the witness field.

   ![Tapscript3](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%843.png?raw=true)

3. The first byte of txins represents the number of transaction inputs, and there is only 1 input in this transaction. When there are multiple inputs, the structure shown below will repeat multiple times. The previous hash represents the first input transaction id, and the outpoint index represents the index in the first input transaction. The size of the unlock script is represented by the first byte after the outpoint index. Since Tapscript must use a witness script, the size of script_sig is 0 and the data is empty. nSequence represents a relative time lock: that is, starting from the previous transaction, the transaction must pass nSequence to take effect.

   ![Tapscript4](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%844.png?raw=true)

4. The first byte of txouts represents the number of transaction outputs, and there is only one output in this transaction. When there are multiple outputs, the structure shown below will repeat multiple times. The value represents the number of BTC received in the first output, `80f0fa0200000000` is also in little-endian mode, the normal value is `0000000002faf080`, and the decimal value is `50000000`. Therefore, the value here means 0.5 BTC. The next byte represents the size of the lock script and the lock script, which represents the address of the recipient. `16` means that the size of the lock script is 22 bytes. The lock script `001482b672c4c6e8f7f06ba7d35950bc6e7925ab3f35`, the first byte `00` has no actual meaning, and corresponds to the opcode OP_0. In the second byte `14`, the decimal is 20, and the corresponding opcode is OP_PUSHBYTES_20, which means that the next 20 bytes will be pushed onto the stack. The remaining 20 bytes represent the public key hash.

   ![Tapscript5](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%845.png?raw=true)

5. Witness represents a witness script. The following shows the witness script corresponding to the first transaction input. In other words, each transaction input has a corresponding witness script. When there are multiple transaction inputs, the structure shown below will repeat multiple times. The first byte of witness indicates the number of elements of the witness script corresponding to the first transaction input. As mentioned above, the original text of the transaction represents a transfer from the Taproot address to an ordinary address, and the Taproot address can be spent in two ways: the aggregated signature of all participants and the spending through the script path. The size of the witness element below is not 1, so instead of using the aggregation signature method, it is spent through the script path. MAST is composed of a single script `[PKA] [OP_CHECKSIG] [PKB] [OP_CHECKSIGADD] [PKC] [OP_CHECKSIGADD] [2] [OP_NUMEQUAL]`, denoted as script_1. Then the corresponding witness script structure should be `Stack element(s) satisfying script_1`, `script_1`, `P`.

   ![Tapscript6](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%846.png?raw=true)
   
   The `Stack element(s) satisfying script_1` in the witness script structure corresponds to the above figure `first_elemnet`, `second_elemnet`, and `third_elemnet`, which represent the signatures of C, B, and A respectively. Since script_1 is 2-3 multi-signature, C did not sign, so the size of `first_elemnet` is 0, `second_elemnet`, `third_elemnet` adopt Schnorr signature, and the fixed size is 64 bytes. `fourth_elemnet` represents script_1, and the specific composition is shown in the figure below: `21` is the opcode OP_PUSHBYTES_21, which means that the next 33 bytes will be pushed onto the stack. `ac` is the opcode OP_CHECKSIG, which means to check the signature. `ba` is the OP_CHECKSIGADD opcode mentioned earlier. `52` is the opcode OP_2, which represents the value 2. `9c` is the opcode OP_NUMEQUAL, used to determine whether two values are equal. The first byte in `fifth_elemnet` represents the Tapscript version number, and the remaining 32 bytes represent the aggregate public key P.
   
   ![Tapscript7](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%847.png?raw=true)

6. nLockTime represents the earliest time that the transaction is allowed to be packaged. There are four cases: 1) 0 means effective immediately. 2) Less than 500000000 represents the block height, and the transaction is locked before the block. 3) Greater than or equal to 500000000 represents the Unix timestamp, and the transaction is locked before that time. 4) When the nSequence fields of the transaction input are all the maximum value `0xffffffff`, the nLockTime field is invalid.

   ![Tapscript8](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%848.png?raw=true)

## Conclusion

From the perspective of transaction structure, Tapscript follows the transaction structure of Segregated Witness, and the main change lies in the update of the public key and signature. If prefixes such as the Tapscirpt version number and parity are ignored, the public key is uniformly represented by 32 bytes.The signature in the witness script uses the Schnorr signature, and the total length is unified to 64 bytes, omitting the redundant bytes in the ECDSA signature.

