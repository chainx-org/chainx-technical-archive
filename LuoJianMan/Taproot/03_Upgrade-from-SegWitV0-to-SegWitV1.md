# Upgrade from SegWit V0 to SegWit V1

## Content

1. [Taproot is a continuation of SegWit](#Taproot%20is%20a%20continuation%20of%20SegWit)
2. [Upgrade from SegWit V0 to SegWit V1](#Upgrade%20from%20SegWit%20V0%20to%20SegWit%20V1)
   1. [SegWit transaction structure](#SegWit%20transaction%20structure)
   2. [Construct SegWit V0 Transaction](#Construct%20SegWit%20V0%20Transaction)
   3. [Construct SegWit V1 Transaction](#Construct%20SegWit%20V1%20Transaction)
3. [Can SegWit V1 compatible with SegWit V0?](#Can%20SegWit%20V1%20compatible%20with%20SegWit%20V0?)
4. [ChainX upgrade Taproot](#ChainX%20upgrade%20Taproot)

## Taproot is a continuation of SegWit

During the quarantine witness *(Segwit)* upgrade, Bitcoin's scripting rules have undergone a large-scale upgrade, proposing two new scripting rules, `P2WPKH` *(Pay To Witness Public Key Hash)* and `P2WSH`*(Pay to Witness Script Hash)*. Addresses starting with `bc1` now adopt one of the above scripting rules. P2WPKH is usually used for ordinary addresses, while P2WSH is usually used for multi-signed addresses. In addition, in the upgrade of isolation witness, a concept of `version number `(Version) is reserved in the script. All previous Segwit script rules use `V0` as the version number, so it can also be called Segwit V0. On the other hand, Taproot is upgraded on the framework of Segwit, and the version number is upgraded to `V1`, which can also be called `P2TR` *(Pay to Taproot)*.

SegWit V0 is divided into P2WPKH (locked to public key hash) and P2WSH (locked to script hash) depending on the locked address, and these two script rules are completely different, so it is easy to distinguish them. However, after upgrading to SegWit V1, this will be difficult to distinguish.

Segwit V1 output can be spent in two ways:
* key path (Key Path) cost: it treats the witness program as a public key and allows you to spend using the signature of that public key.
*  script path (Script Path) cost: it treats the witness program as an execution script, allowing the use of pre-submitted scripts to spend output.

Using the public key and signature aggregation scheme of Musig and spending through the key path (Key Path), the multi-signature transaction of n-of-n can be realized, but the implementation of this method is no different from that of using a single public key.


## Upgrade from SegWit V0 to SegWit V1

### SegWit transaction structure

Segwit v1 follows the same output script pattern as segwit v0:

* Segwit output: **`[1B Version]` `[segwit program]`**
* Segwit v0 output: **`[00]` `[20-Byte public key digest]`** (P2WPKH) or **`[00]` `[32-Byte script digest]`** (P2WSH)
* Segwit v1 output: **`[01]` `[33-Byte public key]`**

The 33-Byte public key encoding is similar to that of legacy compressed pubkeys, but with a different oddness byte.

* Y-coordinate: even - **`[00]`** or odd - **`[01]`**
* X-coordinate: **`[32B x-coordinate]`**
* 33 byte pubkey encoding: **`[1B oddness] [32B x-coordinate]`**

![segwit-v1](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1627463356508-1627463356504.png)

The output can be spent along the **key path** by providing a valid signature for the subkey in the output's scriptPubKey. The spending witness is simply`[sig]`.

The output can be spent along the **script path** if the public key was tweaked with a valid taproot. 

### Construct SegWit V0 Transaction

1. First, generate a key pair, encode the public key using `bip- schnorr` and `bip-taproot pubkey` coding rules, and then encode witness `version` and witness `program` as bech32 addresses. Create a Scriptpubkey, send a BTC to the generated address, and generate a transaction output.

```py
# Generate individual key pairs
privkey1, pubkey1 = generate_key_pair()
privkey2, pubkey2 = generate_key_pair()

# Create the spending script
multisig_script = CScript([CScriptOp(OP_2), pubkey1.get_bytes(), pubkey2.get_bytes(), CScriptOp(OP_2), CScriptOp(OP_CHECKMULTISIG)])

# Hash the spending script
script_hash = sha256(multisig_script)

# Generate the address
version = 0
address = program_to_witness(version, script_hash)
print("bech32 address is {}".format(address))

# Generate coins and create an output
tx = node.generate_and_send_coins(address)
print("Transaction {}, output 0\nsent to {}\n".format(tx.hash, address))
```

2. Manually generate the original transaction text by creating the `CTransaction` object and filling in the required fields.
     * `nVersion`
     * `nLocktime`  
     * `tx_vin` 
     * `tx_vout` 
     * `tx.wit.vtxinwit`
```py
# Create a spending transaction
spending_tx = test.create_spending_transaction(tx.hash)
```


3. Sign the constructed transaction with `ECDSA`, bip-taproot defines the following sighash flags:
    * Legacy sighash flags:
      * `0x01` - **SIGHASH_ALL**
      * `0x02` - **SIGHASH_NONE**
      * `0x03` - **SIGHASH_SINGLE**
      * `0x81` - **SIGHASH_ALL | SIGHASH_ANYONECANPAY**
      * `0x82` - **SIGHASH_NONE | SIGHASH_ANYONECANPAY**
      * `0x83` - **SIGHASH_SINGLE | SIGHASH_ANYONECANPAY**
    * New sighash flag:
      * `0x00` - **SIGHASH_ALL_TAPROOT** same semantics `0x01` **SIGHASH_ALL**

Use `SIGHASH_ ALL` in SegWit V0 and replace it with `SIGHASH_ALL_ TAPROOT` in SegWit V1.

```py
# Generate the segwit v0 signature hash for signing
sighash = SegwitV0SignatureHash(script=multisig_script,
                                txTo=spending_tx,
                                inIdx=0,
                                hashtype=SIGHASH_ALL,
                                amount=100_000_000)

# Sign using ECDSA and append the SIGHASH byte
sig1 = privkey1.sign_ecdsa(sighash) + chr(SIGHASH_ALL).encode('latin-1')
sig2 = privkey2.sign_ecdsa(sighash) + chr(SIGHASH_ALL).encode('latin-1')

print("Signatures:\n- {},\n- {}\n".format(sig1.hex(), sig2.hex()))
```

4. Spend SegWit V0 transaction output (P2WSH)


```py
# Construct witness and add it to the script.
# For a multisig P2WSH input, the script witness is the signatures and the scipt
witness_elements = [b'', sig1, sig2, multisig_script]
spending_tx.wit.vtxinwit.append(CTxInWitness(witness_elements))

print("Spending transaction:\n{}\n".format(spending_tx))
print("Transaction weight: {}\n".format(node.decoderawtransaction(spending_tx.serialize().hex())['weight']))
```

### Construct SegWit V1 Transaction

Compared with SegWit V0 SegWit V1, there are several main differences:

* replace ECDSA signature scheme with Schnorr signature.
* adopt Musig aggregate signature and aggregate public-key scheme.
* improve the signature hash and the opcode of the script

```py
# Generate individual key pairs
privkey1, pubkey1 = generate_key_pair()
privkey2, pubkey2 = generate_key_pair()

# Generate a 2-of-2 aggregate MuSig key using the same pubkeys as before
# Method: generate_musig_key(ECPubKey_list)
c_map, agg_pubkey = generate_musig_key([pubkey1, pubkey2])

# Multiply individual keys with challenges
privkey1_c =  c_map[pubkey1] * privkey1
privkey2_c =  c_map[pubkey2] * privkey2
pubkey1_c =  c_map[pubkey1] * pubkey1 
pubkey2_c =  c_map[pubkey2] * pubkey2

# Create a segwit v1 address for the MuSig aggregate pubkey
# Witness program ([1B oddness] [32B x-coordinate])
pubkey_data = agg_pubkey.get_bytes()
program_musig = bytes([pubkey_data[0] & 1]) + pubkey_data[1:]
print("Musig witness program is {}".format(program.hex()))

version = 0x01
address_musig = program_to_witness(version, program_musig)
print("Musig address is {}".format(address))
print("2-of-2 musig: ", address_musig)

# Generate coins and create an output
tx = node.generate_and_send_coins(address_musig)
print("Transaction {}, output 0\nsent to {}\n".format(tx.hash, address_musig))

# Create sighash for ALL (0x00)
sighash_musig = TaprootSignatureHash(spending_tx, [tx.vout[0]], SIGHASH_ALL_TAPROOT, input_index=0)

# Generate individual nonces for participants and an aggregate nonce point
# Remember to negate the individual nonces if necessary
nonce1 = generate_schnorr_nonce()
nonce2 = generate_schnorr_nonce()
R_agg, negated = aggregate_schnorr_nonces([nonce1.get_pubkey(), nonce2.get_pubkey()])
if negated:
  R_agg.negate()
  nonce1.negate()
  nonce2.negate()

# Create an aggregate signature
# Method: sign_musig(privkey, nonce, R_agg, agg_pubkey, sighash_musig)
# Method: aggregate_musig_signatures(partial_signature_list, R_agg)
s1 = sign_musig(privkey1_c, nonce1, R_agg, agg_pubkey, sighash_musig)
s2 = sign_musig(privkey2_c, nonce2, R_agg, agg_pubkey, sighash_musig)
sig_agg = aggregate_musig_signatures([s1, s2], R_agg)
print("Aggregate signature is {}\n".format(sig_agg.hex()))

# Add witness to transaction
spending_tx.wit.vtxinwit.append(CTxInWitness([sig_agg]))

# Get transaction weight
print("Transaction weight: {}\n".format(node.decoderawtransaction(spending_tx.serialize().hex())['weight']))

print("Success!")
```

SegWit V1 MuSig输出比SegWit V0 P2WSH输出的权重大约低30%，对于更大的n-of-n多重签名交易，交易权重会更加地减少。由于交易费用是基于交易权重，这些权重的节省直接转化为费用的节省。此外，通过使用MuSig聚合密钥和签名，不会透露正在使用的多签名方案，这有利于交易的隐私性和安全性。

## Can SegWit V1 be compatible with SegWit V0?

Overall, SegWit Version 1 has not changed much, mainly in the following aspects:

1. Discard `CHECKMULTISIG`: this opcode is inefficient because it sometimes needs to try the combination of multiple public keys and signatures, and these running logic can be processed more efficiently. Moreover, it is incompatible with batch verification, which verifies all signatures in a block faster than individual verification. However, due to the wide range of applications of the opcode, an opcode `OP_ CHECKSIGADD` is added for replacement. The opcode adds a counter according to the status of the signature check whether it is successful or not.
2. Replace ECDSA with `Schnorr signature`: all opcodes in SegWit Version 0 that use ECDSA signature will be changed to Schnorr signature because Schnorr signature is more efficient. Moreover, ECDSA signatures do not support batch verification, which conflicts with the design goals of bip-taproot.
3. Improved `signature hash`: SegWit Version 1 has made some improvements to the signature hash to solve some long-standing problems.

After upgrading to Taproot, because the upgrade target mentioned in BIP-341 is not compatible with the existing SegWit Version 0 script type, it is not possible to use the script of SegWit Version 0 to spend the output of SegWit Version 1, so the unupgraded node cannot verify the transaction of SegWit Version 1. Even so, this upgrade is necessary and can significantly improve the privacy and security of the Bitcoin network with minor changes.

## ChainX upgrade Taproot

ChainX's current multi-signature scheme will also benefit from activating Taproot. After switching to SwgWit V1, with the integration of various infrastructure, achieving more diversified multiple signatures, more private address types, potential complex logic support, reducing transaction costs, and other aspects of development, all users in the ChainX ecology will eventually be able to benefit from it.