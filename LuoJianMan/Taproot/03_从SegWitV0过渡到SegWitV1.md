# 从SegWit V0过渡到SegWit V1

## 大纲

1. [Taproot是SegWit的延续](#Taproot是SegWit的延续)
2. [从SegWit V0过渡到SegWit V1](#从SegWit V0过渡到SegWit V1)
   1. [SegWit交易结构](#SegWit交易结构)
   2. [构造SegWit V0交易](#构造SegWit V0交易)
   3. [构造SegWit V1交易](#构造SegWit V1交易)
3. [SegWit V1能否兼容SegWit V0？](#SegWit V1能否兼容SegWit V0？)
4. [ChainX多签跨链方案支持Taproot](#ChainX多签跨链方案支持Taproot)

## Taproot是SegWit的延续

在隔离见证*（Segwit）*升级时，比特币的脚本规则进行了一次大规模的升级，提出了两种全新的脚本规则`P2WPKH`  *（Pay to Witness Public Key Hash）*和 `P2WSH` *（Pay to Witness Script Hash）*，现在以 `bc1` 开头的地址，都是采用了上述的脚本规则之一。P2WPKH 通常用在普通的地址上，而P2WSH 通常用在多签地址中。另外，在隔离见证的升级中，脚本中还预留了一个`版本号`（Version）的概念，所有之前的 Segwit 脚本规则都采用了 `V0` 作为版本号，所以也可以称为 Segwit V0。而 Taproot 则是在 Segwit 的框架上进行了升级，版本号升级为 `V1`，也可以称作`P2TR`*(Pay to Taproot)*。

SegWit V0 根据锁定的地址不同，区分为P2WPKH（锁定到公钥哈希）和P2WSH（锁定到脚本哈希），这两种脚本规则完全不同，因此很容易区分开来。但升级至SegWit V1后，这将难以区分。

Segwit V1输出可通过两种方式花费：

 + 密钥路径（Key Path）花费：它将witness program视为公钥，并允许使用该公钥的签名进行花费。
 + 脚本路径（Script Path）花费：它将witness program视为执行脚本，允许使用预先提交的脚本来花费输出。

使用Musig的公钥、签名聚合方案，通过密钥路径（Key Path）进行花费，能够实现n-of-n的多签交易，而这种方式的实现与使用单个公钥进行花费没有区别。

## 从SegWit V0过渡到SegWit V1

### SegWit交易结构

Segwit v1遵循与Segwit v0相同的输出脚本模式:

* Segwit 输出：`[1B Version]` `[SegWit program]`
* Segwit V0 输出: `[00]` `[20B public key digest]`(P2WPKH) 或`[00]` `[32B script digest]`(P2WSH)

* Segwit V1 输出：`[01]` `[33B public key]`

可以看到，在SegWit V1中，33字节的公钥编码与一般的压缩公钥编码类似，不同之处在于字节总数为奇数。

\* Y-坐标: 偶数-`[00]` 或 奇数-`[01]`

\* X-坐标: `[32B x-coordinate]`

\* 33 字节公钥编码：`[1B oddness] [32B x-coordinate]`

![segwit-v1](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1627463356508-1627463356504.png)

通过在输出的scriptPubKey中为pubkey提供有效的签名，可以使用**密钥路径**使用输出，花费的witness只是简单的`sig`（签名），而如果public key 使用了taproot进行调整，则输出可以使用**脚本路径**进行花费。

### 构造SegWit V0（2-2多重签名）

1. 首先生成一个密钥对，使用`bip-schnorr`和`bip-taproot pubkey`编码规则对公钥进行编码，然后将witness `version`和witness `program`编码为bech32地址。创建Scriptpubkey，向生成地址发送btc，生成一笔交易输出

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

2. 通过创建`CTransaction`对象并填充所需字段的方式，手动生成交易原文。
     * `nVersion`
     * `nLocktime`  
     * `tx_vin` 
     * `tx_vout` 
     * `tx.wit.vtxinwit`
```py
# Create a spending transaction
spending_tx = test.create_spending_transaction(tx.hash)
```


3. 对构造的交易进行`ECDSA`签名
  + 先前的签名哈希标志：
     - `0x01` - **SIGHASH_ALL**
     - `0x02` - **SIGHASH_NONE**
     - `0x03` - **SIGHASH_SINGLE**
     - `0x81` - **SIGHASH_ALL | SIGHASH_ANYONECANPAY**
     - `0x82` - **SIGHASH_NONE | SIGHASH_ANYONECANPAY**
     - `0x83` - **SIGHASH_SINGLE | SIGHASH_ANYONECANPAY**
   - 升级Taproot后新增的签名哈希标志
     - `0x00` - **SIGHASH_ALL_TAPROOT** 
     	
     	> 用法类似于`0x01` **SIGHASH_ALL**

在SegWit V0中使用`SIGHASH_ALL`，在SegWit V1中使用`SIGHASH_ALL_TAPROOT`进行替换

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

4. 花费SegWit V0交易输出(P2WSH)


```py
# Construct witness and add it to the script.
# For a multisig P2WSH input, the script witness is the signatures and the scipt
witness_elements = [b'', sig1, sig2, multisig_script]
spending_tx.wit.vtxinwit.append(CTxInWitness(witness_elements))

print("Spending transaction:\n{}\n".format(spending_tx))
print("Transaction weight: {}\n".format(node.decoderawtransaction(spending_tx.serialize().hex())['weight']))
```

### 构造SegWit V1（2-2多重签名）

相较于SegWit V0，SegWit V1主要有以下几点不同：

* 采用Schnorr签名替换ECDSA签名方案
* 采用Musig聚合签名和聚合公钥方案
* 改进签名哈希和脚本的操作码

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

## SegWit Version 1是否能兼容SegWit version 0？

总地来看，SegWit Version 1的改动不大，主要集中在以下几个方面：

1. 「**弃用`OP_CHECKMULTISIG`**」：这个操作码的效率较低，因为它有时需要尝试多个公钥与签名的组合，这些运行逻辑可以被更高效地处理。而且，它与不兼容批量验证（批量验证相较于单独验证能更快地验证一个块中的所有签名）。但由于该操作码应用广泛，新增了一个操作码`OP_CHECKSIGADD`用于替换，该操作码根据签名检查是否成功的状态增加了一个计数器。
2. 「**用Schnorr签名代替ECDSA**」：SegWit Version 0中所有采用ECDSA签名的操作码都将改为采用Schnorr签名，因为Schnorr签名更高效。而且，ECDSA签名不支持批量验证，这与这与bip-taproot的设计目标相冲突。
3. 「**改进签名哈希**」：SegWit Version 1对签名的哈希做了一些改进，以解决一些长期存在的问题。

在升级至Taproot后，由于BIP-341中提到的升级目标与现有的SegWit Version 0脚本类型并不兼容，无法使用SegWit Verion 0的脚本花费SegWit Version 1的输出，因此，未升级的节点是无法验证SegWit Version 1的交易的。虽然如此，但是这个升级是必要的，能够通过不大的改动显著提升比特币网络的隐私性和安全性。

## ChainX多签跨链方案支持Taproot

ChainX当前的多重签名方案也会因为激活Taproot后而受益，在切换到SwgWit V1后，随着各种基础设施的集成，实现更多样化的多签、私密性更强的地址类型、潜在的复杂逻辑支持、降低交易成本等各个方面的发展，最终ChainX生态内的所有用户都将能够从中收益。