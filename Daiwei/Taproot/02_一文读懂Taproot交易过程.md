# 一文读懂Taproot交易过程

Taproot的核心是由Schnorr 签名和MAST抽象语法树组成，Taproot交易过程本文主要从Taproot公钥的创建和Taproot的花费模式两个方面阐述。



## 创建Taproot公钥

为了创建Taproot公钥，首先需要了解Taproot聚合公钥和聚合签名的产生过程。目前关注度较高的相关研究包括MuSig1和MuSig2。相比于MuSig2两轮通信，MuSig1最大的缺点在于它创建签名需要三轮通信，而每一轮通信都由来回传递的消息组成。由于本文未涉及网络间通信，因此这里主要讨论的是聚合公钥和聚合签名的基本生成过程。

### 聚合公钥

![test](https://github.com/bitcoinops/taproot-workshop/blob/Colab/images/musig_intro_1.jpg?raw=1)

从上图可以看出聚合公钥的生成过程可以分三步：1）相互传递公钥，并对所有公钥拼接进行一次聚合hash，生成c_all。  2）c_all和i的公钥拼接hash后生成因子c_i。  3) 根据因子c_i进行线性组合得到聚合公钥P_agg。

![image-20210722091644213](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722091644213.png?raw=true)

### 聚合签名

![test](https://github.com/bitcoinops/taproot-workshop/blob/Colab/images/musig_intro_2.jpg?raw=1)

聚合签名的生成过程上图显示需要进行三轮通信，主要分成两大步：1）生成nonce并线性聚合生成R_agg。  2）各方利用私钥生成Schnorr签名并进行聚合，得到最终的聚合签名（R_agg, s1+s2+s3）。

![image-20210722091724750](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722091724750.png?raw=true)

### 引入MAST抽象语法树

![img](https://github.com/bitcoinops/taproot-workshop/blob/master/images/taptree0.jpg?raw=true)

如上图所示，Taproot公钥主要由两部分组成，包括聚合公钥P和MAST结构形成的公钥tG。假设P是Alice,Bob,Charlie的聚合公钥，script_A,script_B,script_C是Alice,Bob,Charlie相关的脚本。那么，Taproot公钥创建过程如下：

1. Alice,Bob和Charlie各自生成公私钥。

   ![image-20210722092748689](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092748689.png?raw=true)

2. 公钥聚合成pubkey_agg，并调整私钥，用于以后的签名。

   ![image-20210722092816042](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092816042.png?raw=true)

3. 创建脚本script_A,script_B,script_C

   ![image-20210722092854675](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092854675.png?raw=true)

4. 构建MAST抽象语法树，计算MAST结构对应的私钥taptweak。上图中TaggedHash表示带标签的hash,固定长度32字节，计算方式是TaggedHash(tag, x) = sha256(sha256(tag) + sha256(tag) + x)；ver表示Tapscript版本号，当前值为0xc0；size表示scirpt的字节数；A&B表示A,B字典排序后按字节拼接。

   ![image-20210722092920158](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092920158.png?raw=true)

4. 根据公式Q=P+tG合成taproot公钥,并生成segwit_address用于下文交易。

   ![image-20210722092943482](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722092943482.png?raw=true)

5. 向Taproot地址转账50个btc

   ![image-20210722093015597](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093015597.png?raw=true)

## Taproot花费

为了完成从上述Taproot地址向Bob个人转账0.5个btc,有两种支付途径：一种是Alice，Bob，Charlie都进行签名后形成聚合签名，完成向Bob的转账；另一种是通过MAST结构中的script向Bob进行转账。

1. 创建交易原文，填充接受方地址，转账数量等数据。

   ![image-20210722093042214](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093042214.png?raw=true)

2. 按照第一种方式进行转账，首先需要Alice，Bob，Charlie各自生成nonce并进行聚合，然后各自利用私钥进行Schnorr签名，最后进行签名的聚合操作。因此，这种方式最终的见证脚本是一个单独的签名,固定长度64字节。

   ![image-20210722093114659](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093114659.png?raw=true)

3. 测试第一种花费组建的交易原文合法并发送交易

   ![image-20210722093202683](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093202683.png?raw=true)

4. 按照第二种方式进行转账，假设是Alice通过script_A完成向Bob的转账，那么见证脚本需要包括：1）`[Stack element(s) satisfying TapScript_A]` 2）`[TapScript_A]` 3）`[Controlblock c]`。其中`[Controlblock c]`表示的是`TapScript_A`相关的proof,长度为33+32n。33个字节中的首字节是聚合公钥和Taproot版本号共同计算得出，剩下32个字节表示的是聚合公钥的x坐标。32n表示的是`TapScript_A`的proof，在本例中n=2，指的是`taggedhash_leafB`和`taggedhash_leafC`。

   ![image-20210722093350335](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093350335.png?raw=true)

5. 测试第二种花费组建的交易原文合法并发送交易

   ![image-20210722093409770](https://github.com/AAweidai/PictureBed/blob/master/taproot/image-20210722093409770.png?raw=true)

## 总结

总的来看，Taproot交易主要关注的是一种输出和两种花费模式。一种输出使得无论是个人交易还是多签交易，锁定脚本中公钥是一致的，无法从形式上进行区分。两种花费模式使得我们可以用更少的字节，实现更复杂的交易过程。