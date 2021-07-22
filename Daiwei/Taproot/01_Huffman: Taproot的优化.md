# Huffman: Taproot的优化

## Taproot回顾

​	比特币改进提案（BIPs）是为比特币引入新功能和信息的设计文档，而 Taproot 升级则是三个 BIPs 的汇编，这三个 BIPs 分别是 Schnorr 签名(BIP 340)、Taproot (BIP 341)和TapScript (BIP 342)，这三个升级统称为 BIP Taproot，它将为比特币带来了更高效、更灵活、更私密的传输方式，其核心在于使用了 Schnorr 签名和 Merkel 抽象语法树(MAST)。

![img](https://miro.medium.com/max/1400/0*d2isTQT49nyPldLo)

Taproot的原理，简单来说，就是定义了一种输出和两种花费路径。如上图所示，有Alice和Bob两个参与者，Taproot的运作过程如下:

- 将Alice和Bob的公钥聚合为： C=P_A+P_B 
- 加入MerkleRoot，公钥聚合为：P=C+H(C||MerkleRoot)G，其中 H(C||MerkleRoot) 表示C和MerkleRoot的组合hash

- 锁定脚本中填入聚合公钥P，格式类似Segwit：`[ver] [P]`​。ver表示版本号，Taproot中ver=1。
- 花费路径有两个，二选一：
  1. 签名模式：Alice和Bob全部签名产生聚合签名，填入见证脚本。利用聚合公钥P,对签名进行验证后即可花费。
  2. 脚本模式：Alice和Bob，有一个拒绝签名，可以走脚本模式。此时Alice想要完成花费，那么见证脚本中需要填入：`符合Script 1的执行数据D, Script 1, C, Hash 2 `。为了验证签名，首先利用`Script 1, Hash2`，计算MerkleRoot，然后验证 P==C+H(C||MerkleRoot)G  是否成立，最后构建完整脚本`D||Script 1`并执行脚本，验证结果是否为真。当上述验证通过后，即可完成花费。

​	Taproot按照签名模式进行花费时，多个参与方和单个参与方在区块链上看起来都长得是一样的，所以许多区块链分析将不可用，从而为所有 Taproot 用户保留隐私。与此同时，Taproot的脚本花费模式让用户可以实现复杂的支出条件。Taproot可以有效压缩交易脚本字节数。签名模式下随着参与者增加，EDSA交易脚本大小线性增长，Taproot交易脚本大小保持不变。脚本模式下随着脚本数量的增加，EDSA交易脚本大小线性增长，Taproot交易脚本大小对数增长。

# Huffman Tree

​	给定N个权值作为N个叶子结点，构造一棵二叉树，若该树的带权路径长度达到最小，称这样的二叉树为最优二叉树，也称为哈夫曼树(Huffman Tree)。哈夫曼树是带权路径长度最短的树，权值较大的结点离根较近。

![img](https://github.com/bitcoinops/taproot-workshop/blob/master/images/huffman_intro0.jpg?raw=true)

​	哈夫曼树的构造过程是利用贪心的思想，每次选择权重最小的两个结点组合成新的节点，加入到原始集合，重复上述过程，直至结束。如上图所示，存在权重集合为 (1, 2, 3, 4, 5)  的5个结点，构造哈夫曼树的过程如下：

- 从集合中选择权重最小的 1, 2 结点，组合成权重为3的结点，集合变成 (3,3,4,5) 
- 从集合中选择权重最小的 3, 3 结点，组合成权重为6的结点，集合变成 (4, 5, 6) 
- 从集合中选择权重最小的 4, 5 结点，组合成权重为9的结点，集合变成 (6, 9) 
- 从集合中选择权重最小的 6, 9 结点，组合成权重为15的结点，集合变成 (15) , 构造过程结束

可以看出，在哈夫曼树中，权重越大的结点离根越近，权重越小的结点离根越远。

# Huffman Tree在Taproot的应用

​	Taproot相比于ECDSA多签算法，一大优势是Schnorr聚合签名可以将多个签名数据压缩合并成单个聚合签名。使用聚合签名技术后，交易脚本的字节数大大减少，可以有效减少交易费用的支出。当我们按照脚本模式进行花费时，借助MAST的树形结构，交易脚本大小随着脚本数量对数增长。然而利用Huffman Tree，可以通过改进MAST的构造过程，帮助我们进一步减少脚本花费模式下见证脚本的字节数，最终达到降低交易手续费的目的。

![img](https://github.com/bitcoinops/taproot-workshop/blob/master/images/taptree0.jpg?raw=true)

​	前文提到，当Taproot按照脚本模式进行花费时，需要提供脚本在MAST中的Proof。如上图，当我们想通过script_A进行花费，见证脚本中需要提供：

- 符合script_A的执行数据D
- script_A
- P
-  TaggedHash(B), TaggedHash(C)  TaggedHash表示带标签的hash，固定长度32字节，使用带标签的hash目的是减少hash碰撞

​	忽略前两项的字节数，可以看出，最后一项的字节数可以表示为 32*(d-1) ，d=3表示 script_A在MAST中的高度。换句话说，最后一项的字节数与脚本在MAST中高度线性相关。因此，对于 script\_A, script\_B, script\_C 三个脚本，我们可以根据脚本使用频率赋予权重，构造哈夫曼树，作为MAST。最终，根据Huffman的特性，使用频率越高的 script , 在哈夫曼树中的高度越低，见证脚本中字节数越少，交易手续费越低。

​	总而言之，交易手续费与交易脚本的字节数息息相关。Taproot的聚合签名可以帮助我们有效减少按签名模式花费时的交易脚本字节数，但是利用Huffman构造MAST可以进一步帮助我们减少按脚本模式花费时的交易脚本字节数。

