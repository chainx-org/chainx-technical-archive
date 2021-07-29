## Content Guide

- [Script Path Spending](#Script-Path-Spending)
- [Build Taptree](#Build-Taptree)
  - [Build Taptree directly and calculate taptweak](#Build-Taptree-directly-and-calculate-taptweak)
  - [Build Taptree by Taproot Descriptor](#Build-Taptree-by-Taproot-Descriptor)
    - [Constructing a taptree from a descriptor](#Constructing-a-taptree-from-a-descriptor)
- [Spend along the script path](#Spend-along-the-script-path)
  - [Constructing a taproot output from a taptree](#Constructing-a-taproot-output-from-a-taptree)
  - [Sign the transaction for `TapLeafA`](#Sign-the-transaction-for-)
  - [Construct the witness, add it to the transaction and verify mempool acceptance](#Construct-the-witness,-add-it-to-the-transaction-and-verify-mempool-acceptance)

# Script Path Spending

**Script path spending**, allowing the use of pre-submitted scripts to spend Taproot output. Spending through the script path first needs to build the script path. Here we build the script path through Taptree. Through Taptree, we can submit multiple Tapscripts to one tapweak (Taproot uses tweak as a commitment to spend the script path).

## Build Taptree
Submitting multiple Tapscripts requires a submission structure similar to the Merkel tree structure. **TapTree differs from the head Merkel tree in the following aspects:**

* Tapleaves can be located at different heights
* Ordering of TapLeaves is determined lexicograpically
* Location of nodes are tagged (No ambiguity of node type)

The internal nodes of Taptree (non-tapscript nodes or auxiliary nodes) are called tapbranch and are calculated using the `tagged_hash("Tag", input_data) `function. The tag hash is very useful when constructing Taptree promises. They can prevent the high ambiguity of the nodes currently found in the Merkel tree of the transaction, which allows an attacker to create a node that can be reinterpreted as a leaf node or an internal node, and the tag hash can ensure that a tapleaf is not misunderstood as an internal node. Node and vice versa.

![Tagged Hashes](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627374446982-1627374446972.png)

#### Build Taptree directly and calculate taptweak
1. Calculate TapLeaves A, B, and C
2. Calculate internal TapBranch AB
3. Calculation TapTweak
4. Export bech32m address

```python
TAPSCRIPT_VER = bytes([0xc0])  
internal_pubkey = ECPubKey()
internal_pubkey.set(bytes.fromhex('03af455f4989d122e9185f8c351dbaecd13adca3eef8a9d38ef8ffed6867e342e3'))

# Derive pay-to-pubkey scripts
privkeyA, pubkeyA = generate_bip340_key_pair()
privkeyB, pubkeyB = generate_bip340_key_pair()
privkeyC, pubkeyC = generate_bip340_key_pair()
scriptA = CScript([pubkeyA.get_bytes(), OP_CHECKSIG]) 
scriptB = CScript([pubkeyB.get_bytes(), OP_CHECKSIG])
scriptC = CScript([pubkeyC.get_bytes(), OP_CHECKSIG])

# Method: Returns tapbranch hash. Child hashes are lexographically sorted and then concatenated.
# l: tagged hash of left child
# r: tagged hash of right child
def tapbranch_hash(l, r):
    return tagged_hash("TapBranch", b''.join(sorted([l,r])))

# 1) Compute TapLeaves A, B and C.
# Method: ser_string(data) is a function which adds compactsize to input data.
hash_inputA = ser_string(scriptA) 
hash_inputB = ser_string(scriptB) 
hash_inputC = ser_string(scriptC) 
taggedhash_leafA = tagged_hash("TapLeaf", TAPSCRIPT_VER + hash_inputA) 
taggedhash_leafB = tagged_hash("TapLeaf", TAPSCRIPT_VER + hash_inputB) 
taggedhash_leafC = tagged_hash("TapLeaf", TAPSCRIPT_VER + hash_inputC) 

# 2) Compute Internal node TapBranch AB.
internal_nodeAB = tapbranch_hash(taggedhash_leafA, taggedhash_leafB)

# 3) Compute TapTweak.
rootABC =  tapbranch_hash(internal_nodeAB, taggedhash_leafC)
taptweak =  tagged_hash("TapTweak", internal_pubkey.get_bytes() + rootABC)
print("TapTweak:", taptweak.hex())

# 4) Derive the bech32m address.
taproot_pubkey_b = internal_pubkey.tweak_add(taptweak).get_bytes()
bech32m_address = program_to_witness(1, taproot_pubkey_b)
print('Bech32m address:', bech32m_address)
```

```json
TapTweak: 12c7dc67ad990309d0ccbec1146615dbc00b748fd2f9c73a98161433aa729806
Bech32m address: bcrt1p00dclafqfx35eh228rqqv337kkfyalxcj8yvymq5rflj4eldmuqswtq26f
```

#### Build Taptree by Taproot Descriptor

For taproot, there is also a taproot descriptor expression, which can be composed of its individual tapscripts. The structure of Taptree is not unique to a set of Tapscripts, so it must also be captured by the Taproot descriptor. Consider the following example, which contains 5 ts(pk(key)) tapscripts.

![Taproot Descriptor](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627374795680-1627374795677.png)

The Taproot descriptor includes:
* `tp(internal_key, [tapscript, [tapscript', tapscript'']])`
* `tp(internal_key, [tapscript])` for single tapscript commitments
* Each node is represented as a tuple of its children, and can be nested within other node expressions
* The left or right ordering of the children is not unique, since they are ultimately ordered lexicographically when computing the taptweak

#### Constructing a taptree from a descriptor

![taptree with 3 leaves](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627374887762-1627374887757.png)

In this example, we will construct the taptree shown in the descriptor string above. This can be conveniently done by parsing the descriptor string.
```python
# Generate internal key pairs
privkey_internal, pubkey_internal = generate_bip340_key_pair()
pk_hex = pubkey_internal.get_bytes().hex()

# Construct descriptor string
ts_desc_A = 'ts(pk({}))'.format(pubkeyA.get_bytes().hex())
ts_desc_B = 'ts(pk({}))'.format(pubkeyB.get_bytes().hex())
ts_desc_C = 'ts(pk({}))'.format(pubkeyC.get_bytes().hex())
tp_desc = 'tp({},[[{},{}],{}])'.format(pk_hex,
                                       ts_desc_A,
                                       ts_desc_B,
                                       ts_desc_C)
print("Raw taproot descriptor: {}\n".format(tp_desc))

# Generate taptree from descriptor
taptree = TapTree()
taptree.from_desc(tp_desc)

# This should match the descriptor we built above
assert taptree.desc == tp_desc

# Compute taproot output
taproot_script, tweak, control_map = taptree.construct()

print("Taproot script hex (Segwit v1):", taproot_script.hex())
```

```json
Raw taproot descriptor: tp(3fbff70d3b52dd1cf2feca1c979fde93736e625b7a272eac8ff7201bec76a581,[[ts(pk(4871ead4bb967ad3bd480e6b5d0d97e430587d5d3fc71e0acac2901c417b0ffb)),ts(pk(6156b94325d25f760f024bb9625a248f0b53e410d872085daf8905f066f14343))],ts(pk(6971b327545b59a5f914834c27f35f74e7eaae3433bd0df364612859d4a49b2e))])

Taproot script hex (Segwit v1): 512032e5c4822af5336efe5f751c71ebc7aae4383e381866f0e1d9931c3adcc91e4e
```

## Spend along the script path

A Taproot output is spent along the script path with the following witness pattern:
* Witness to spend TapScript_A:
  * ``[Stack element(s) satisfying TapScript_A]``
  * `[TapScript_A]`
  * ``[Controlblock c]``

Compared to the script spend path of a taproot with a single committed tapscript, the controlblock spending a taproot containing multiple tapscripts will also include a script inclusion proof.

* Controlblock c contains:
  * `[Tapscript Version]` 
    * `0xfe & c[0]`
  * ``[Parity bit (oddness of Q's y-coordinate)]``
    * `0x01 & c[0]`
  * `[Internal Public Key]` 
  * `c[1:33]`
  * `[Script Inclusion Proof]` 
    * `n x 32Bytes`

Note that this script inclusion proof is a 32B multiple and its size will depend on the position of tapscript in the taptree structure.

![Tapscript inclusion proof](https://cdn.jsdelivr.net/gh/hacpy/PictureBed@master/Document/1627375186235-1627375186231.png)

**Generating the Controlblock**
We use the the taptree construct method to generate the taproot output, tweak and controlblocks for all tapscripts.
**`TapTree.construct()`**returns the tuple:

* `taproot_output_script`、`tweak`、`control_block_map`
* `control_block_map` has key-value pairs: 
  * `tapscript.script` - `controlblock`

#### Constructing a taproot output from a taptree

```python
# Generate key pairs for internal pubkey and pay-to-pubkey tapscripts
privkey_internal, pubkey_internal = generate_bip340_key_pair()

privkeyA, pubkeyA = generate_bip340_key_pair()
privkeyB, pubkeyB = generate_bip340_key_pair()
privkeyC, pubkeyC = generate_bip340_key_pair()
privkeyD, pubkeyD = generate_bip340_key_pair()

# Construct pay-to-pubkey tapleaves and taptree
TapLeafA = TapLeaf().construct_pk(pubkeyA)
TapLeafB = TapLeaf().construct_pk(pubkeyB)
TapLeafC = TapLeaf().construct_pk(pubkeyC)
TapLeafD = TapLeaf().construct_pk(pubkeyD)

# Create a taptree with tapleaves and huffman constructor
# Method: TapTree.huffman_constructor(tuple_list)
taptree =  TapTree(key=pubkey_internal)
taptree.huffman_constructor([(2, TapLeafA), (2, TapLeafB), (2, TapLeafC), (2, TapLeafD)])

# Generate taproot tree with the `construct()` method, then use the taproot bytes to create a bech32m address
taproot_script, tweak, control_map = taptree.construct()
taproot_pubkey = pubkey_internal.tweak_add(tweak) 
program = taproot_pubkey.get_bytes()
address = program_to_witness(1, program)
print("Address: {}".format(address))
```

```json
Address: bcrt1pfnqgwf7ug0vfqrhva9r5s7m8thunak0xxeld95tlqy77y8zxfm9qsd03hd
```

#### Sign the transaction for `TapLeafA`

Note that we must pass the following arguments to `TaprootSignatureHash` for script path spending:
* `scriptpath`：`True`
* `script`：`Cscript` of tapscript

```python
# Generate the taproot signature hash for signing
sighashA = TaprootSignatureHash(spending_tx,
                               [tx.vout[0]],
                               SIGHASH_ALL_TAPROOT,
                               input_index=0,
                               scriptpath=True,
                               script=TapLeafA.script)

signatureA = privkeyA.sign_schnorr(sighashA)

print("Signature for TapLeafA: {}\n".format(signatureA.hex()))
```

```json
Signature for TapLeafA: afb4d954800448e7c6b5d64d9bba45a980a1b5caaa2e911b31f8b8e9c814379bd0aa836cec90e8038f3b4dafd6aaf8c8a12ce03714748fc55088bcaf27b5512c
```

#### Construct the witness, add it to the transaction and verify mempool acceptance

```python
# Add witness to transaction
# Tip: Witness stack for script path - [satisfying elements for tapscript] [TapLeaf.script] [controlblock]
# Tip: Controlblock for a tapscript in control_map[TapLeaf.script]
witness_elements = [signatureA, TapLeafA.script, control_map[TapLeafA.script]]
spending_tx.wit.vtxinwit.append(CTxInWitness(witness_elements))

# Test mempool acceptance
assert node.test_transaction(spending_tx)
print("Success!")
```

```json
{'txid': '586a0d10863c1a5ca33d7761bdedd328a30aed654d2e814f2babc6f220c0f834', 'wtxid': '0e0eba9c507a94da793a4f107866097ac3e018f7c320969622e270d3b38c9647', 'allowed': True, 'vsize': 133, 'fees': {'base': Decimal('0.50000000')}}
Success!
```