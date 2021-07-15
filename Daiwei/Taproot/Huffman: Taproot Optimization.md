# Huffman: Taproot Optimization

## Taproot Review

​	Bitcoin Improvement Proposals (BIPs) are design documents that introduce new features and information to Bitcoin. Taproot upgrades are a compilation of three BIPs. These three BIPs are Schnorr signatures (BIP 340), Taproot (BIP 341),  TapScript (BIP 342). These three upgrades are collectively referred to as BIP Taproot, which will bring a more efficient, flexible, and more private transmission method for Bitcoin. Its core lies in the use of Schnorr signatures and Merkel abstract syntax trees (MAST).

![img](https://miro.medium.com/max/1400/0*d2isTQT49nyPldLo)

​	The principle of Taproot, in simple terms, is to define one output and two cost paths. As shown in the figure above, there are two participants, Alice and Bob, and the operation process of Taproot is as follows:

- Aggregate the public keys of Alice and Bob into C=P_A+P_B
- Join MerkleRoot, the public key aggregation is:  P=C+H(C||MerkleRoot)G，where  H(C||MerkleRoot)  represents the combined hash of C and MerkleRoot

- Fill in the aggregate public key P in the locking script, which the format is similar to Segwit: `[ver] [P]` ver represents the version number and ver=1 in Taproot.
- There are two cost paths. Choose one of the two:
  1. Signature mode: Alice and Bob all sign to generate an aggregate signature and fill in the witness script. By using the aggregated public key P, it can be spent after verifying the signature.
  2. Script mode: Alice and Bob, who have a rejection signature, can go in script mode. At this point, Alice wants to complete the cost, so the witness script needs to be filled in: `Execution data D that meets Script 1, Script 1, C, Hash 2 `. In order to verify the signature, first use  `Script 1, Hash2` to calculate MerkleRoot. Then verify whether  P==C+H(C||MerkleRoot)G  is established. Finally, build a complete script  `D||Script 1`  and execute the script to verify whether the result is true. If the above verification is passed, the spending can be completed.

​	When Taproot spends according to the signature mode, multiple participants and a single participant look the same on the blockchain, so many blockchain analyses will be unavailable, thereby preserving privacy for all Taproot users. At the same time, the Taproot script spending model allows users to implement complex spending conditions. The Taproot can effectively compress the bytes of trading scripts. In the signature mode, as the number of participants increases, the size of the EDSA transaction script increases linearly, and the size of the Taproot transaction script remains unchanged. In script mode, as the number of scripts increases, the size of EDSA transaction scripts increases linearly, and the size of Taproot transaction scripts increases logarithmically.

# Huffman Tree

​	Given N weights as N leaf nodes, a binary tree is constructed. If the weighted path length of the tree reaches the minimum, such a binary tree is called an optimal binary tree, also called a Huffman tree. The Huffman tree is the tree with the shortest weighted path length, and the node with the larger weight is closer to the root.

![img](https://github.com/bitcoinops/taproot-workshop/blob/master/images/huffman_intro0.jpg?raw=true)

​	The construction process of the Huffman tree is to use the idea of greed. Each time the two nodes with the smallest weight are selected and combined into a new node added to the original set, and the above process is repeated until the end. As shown in the figure above, there are five nodes with a weight set of  (1, 2, 3, 4, 5). The process of constructing a Huffman tree is as follows:

- Choose the  1, 2 nodes with the smallest weight from the set, combine it into a node with a weight of 3, and the set becomes (3,3,4,5) 
- Choose the  3, 3 nodes with the smallest weight from the set, combine it into a node with a weight of 6, and the set becomes  (4, 5, 6) 
- Choose the  4, 5 nodes with the smallest weight from the set, combine it into a node with a weight of 9, and the set becomes  (6, 9) 
- Choose the  6, 9 nodes with the smallest weight from the set, combine it into a node with a weight of 15, the set becomes  (15), and the construction process ends

​	It can be seen that in the Huffman tree, the node with the larger weight is closer to the root, and the node with the smaller weight is farther away from the root.

# Application of Huffman Tree in Taproot

​	Compared with the ECDSA multi-signature algorithm, Taproot has a major advantage in which the Schnorr aggregate signature can be compressed and merge multiple signature data into a single aggregate signature. After the use of aggregate signature technology, the number of bytes of transaction scripts is greatly reduced, which can effectively reduce transaction costs. When we spend according to the script mode, with the help of the tree structure of MAST, the transaction script size increases with the logarithm of the number of scripts. However, the use of the Huffman Tree can help us to further reduce the number of bytes of the witness script in the script spending mode by improving the MAST construction process, and ultimately achieve the purpose of reducing transaction fees.

![img](https://github.com/bitcoinops/taproot-workshop/blob/master/images/taptree0.jpg?raw=true)

​	As mentioned above, when Taproot spends according to the script mode, it needs to provide the Proof of the script in MAST. As shown above, when we want to spend through script_A, we need to provide in the witness script:

- Execution data D that meets script_A
- script_A
- P
-  TaggedHash(B), TaggedHash(C): TaggedHash means tagged hash, with a fixed length of 32 bytes. The purpose of using a tagged hash is to reduce hash collisions

​	If ignoring the number of bytes in the first two items, it can be seen that the number of bytes in the last item can be expressed as  32*(d-1), d=3 indicates the height of script_A in MAST. In other words, the number of bytes in the last item is highly linearly related to the script in MAST. Therefore, for the three scripts of  script\_A, script\_B, script\_C, we can assign weights according to the frequency of script used to construct a Huffman tree as MAST. Finally, according to the above Huffman characteristics, the more frequently used script, the lower the height in the Huffman tree, the fewer bytes in the witness script, and the lower the transaction fee.

​	In conclusion, the transaction fee is closely related to the number of bytes of the transaction script. Taproot’s aggregate signature can help us effectively reduce the number of transaction script bytes spent in signature mode, but using Huffman to construct MAST can further help us reduce the number of transaction script bytes spent in script mode.

