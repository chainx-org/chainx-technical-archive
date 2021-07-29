# Tapscript交易结构

## Tapscript

Taproot 升级是三个 BIPs 的汇编，这三个 BIPs 分别是 Schnorr 签名(BIP 340)、Taproot (BIP 341)和TapScript (BIP 342)。 而Tapscript是Taproot升级的第三部分，是对 Schnorr 和 Taproot 进行的补充。Tapscript主要是对升级机制和操作码进行了改进。

Tapscript 通过使用操作码 OP_SUCCESS可以轻松实现不可预见的未来升级。操作码（Opcode）是比特币脚本语言的运算符，Tapscript 保留了适用于 v0 见证脚本的大部分操作码，对少量操作码进行了更新。Tapscript交易结构的变化主要来自于锁定脚本和见证脚本，本文将通过一个具体例子介绍Tapscript交易各个字段的组成。

## 操作码

Tapscript中操作码的改变有：1）`OP_CHECKSIG/OP_CHECKSIGVERIFY`操作码由原来的验证ECDSA签名变成验证Schnorr签名；2）禁用`OP_CHECKMULTISIG/ OP_CHECKMULTISIGVERIFY`这两个多签操作码；3）增加操作码`OP_CHECKSIGADD`，取代原来的多签操作码。

由于签名验证是比特币脚本中 CPU 最密集的操作，因此上述操作码变化对于实现基于 Schnorr 多重签名方案相关的效率提升至关重要。`OP_CHECKSIG/OP_CHECKSIGVERIFY`的改变自然是为了Taproot 升级中Schnorr签名能够使用。`OP_CHECKSIGADD`这种多重签名操作码通过要求见证人为每个公钥提供有效或无效的签名，从而避免在 k-of-n 多重签名脚本中浪费每个公钥的签名验证操作。

传统的2-of-3多重签名交易脚本如下：

- script:`2 [PK0] [PK1] [PK2] 3 OP_CHECKMULTISIG`
- 可能的见证脚本：`[sig2]` `[sig1]`

使用 Tapscript，实现相同的多重签名策略，其中脚本是:

- Tapscript: `[PK0] [OP_CHECKSIG] [PK1] [OP_CHECKSIGADD] [PK2] [OP_CHECKSIGADD] [2] [OP_NUMEQUAL]`
- 可能的见证脚本:`[sig2]` `[sig1]` `[]`

从上述脚本中可以看出，传统的多重签名`sig`和`PK`的映射关系未知。对于签名`sig1`来说，需要与`PK0`,`PK1`, `PK2`分别进行一次轮询验证才能确定`sig1`是否验证通过，`sig2`同理。而Tapscript为每一个公钥都提供了有效或无效的签名，签名有效时，通过OP_CHECKSIGADD操作码进行累加，最后只需验证有效签名的数量是否达到阈值判断签名是否通过。OP_CHECKSIGADD省略了传统多签验证中的轮询过程，加快了多签验证的速度。

## 交易结构

Tapscript沿用隔离见证中的交易结构，由下面几个字段组成。Tapscript规定交易输入必须是隔离见证花费。也就意味着txins中的script_sig一般为空，而witness字段存在数据。本文通过一个具体的例子来说明各个字段的内涵。

![Tapscript1](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%841.png?raw=true)

1. 下文是一个Taproot地址转账到普通地址的交易原文。交易原文是16进制表示，每两个字符表示一个字节。Taproot地址由聚合公钥和MAST结构对应的公钥组成：Q=P+tG。这里P表示A,B,C三人的聚合公钥，计算公式`P=pubkey_agg(PKA, PKB, PKC)`。tG表示单独的脚本`[PKA] [OP_CHECKSIG] [PKB] [OP_CHECKSIGADD] [PKC] [OP_CHECKSIGADD] [2] [OP_NUMEQUAL]`作为MAST的root计算得到的公钥,计算公式`t=tagged_hash(P||root)`。

   ![Tapscript2](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%842.png?raw=true)

2. 如下图所示，第一行表示字段名字，`()`中的数字表示对应字段的字节数，第二行表示字段在上述交易原文中的对应数据。nVersion表示交易结构的版本标识。`02000000`是小端模式，低位字节在前，因此实际值为`00000002`。 marker是一个值必须为`00`的字段，flag表示是否存在见证脚本，也就是witness字段是否存在数据。

   ![Tapscript3](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%843.png?raw=true)

3. txins第一个字节表示交易输入的个数，该交易中只有1个输入。当有多个输入时，下图结构会重复出现多次。previous hash表示的第一个输入的交易id，outpoint index表示在第一个输入交易中的索引。outpoint index之后第一个字节表示的解锁脚本的大小。由于Tapscript必须采用见证脚本，因此script_sig的大小为0，数据为空。nSequence表示的是相对时间锁：即从上一个交易开始，必须经过nSequence的时间交易才能生效。

   ![Tapscript4](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%844.png?raw=true)

4. txouts第一个字节表示交易输出的个数，该交易中只有一个输出。当有多个输出时，下图结构会重复出现多次。value表示的是第一个输出收到的btc数量，`80f0fa0200000000`同样是小端模式，正常值为`0000000002faf080`,转化成十进制值为`50000000`。因此这里的value表示0.5个btc。接下来的字节表示锁定脚本的大小和锁定脚本，代表接收方的地址。`16`表示锁定脚本的大小是22字节。锁定脚本`001482b672c4c6e8f7f06ba7d35950bc6e7925ab3f35`，第一个字节`00`没有实际含义，对应操作码OP_0。第二个字节`14`，十进制为20，对应操作码为OP_PUSHBYTES_20，表示将接下来20个字节入栈。剩余的20个字节表示公钥hash。

   ![Tapscript5](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%845.png?raw=true)

5. witness表示见证脚本，下文表示的是第一个交易输入对应的见证脚本，换句话说，每一个交易输入都有与之对应的见证脚本。当有多个交易输入时，下图结构会重复出现多次。witness第一个字节表示第一个交易输入对应的见证脚本的元素个数。前文提到交易原文代表Taproot地址向普通地址转账，而Taproot地址的花费有两种方式：所有参与方的聚合签名和通过脚本路径进行花费。下文见证元素大小不为1，因此不是采用聚合签名的方式，而是通过脚本路径进行花费。MAST由单个脚本`[PKA] [OP_CHECKSIG] [PKB] [OP_CHECKSIGADD] [PKC] [OP_CHECKSIGADD] [2] [OP_NUMEQUAL]`组成，记做script_1。那么与之对应的见证脚本结构应为`Stack element(s) satisfying script_1`, `script_1`,`P`。

   ![Tapscript6](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%846.png?raw=true)
   
   见证脚本结构中`Stack element(s) satisfying script_1`对应上图`first_elemnet`，`second_elemnet`，`third_elemnet`，分别代表C, B, A的签名。由于script_1是2-3多签，C没有进行签名，因此`first_elemnet`大小为0，`second_elemnet`,`third_elemnet`采用Schnorr签名，固定大小64字节。`fourth_elemnet`代表script_1，具体组成如下图所示：`21`是操作码OP_PUSHBYTES_21 ，表示将接下来33个字节入栈。`ac`是操作码OP_CHECKSIG，表示检查签名。`ba`是前文提到的OP_CHECKSIGADD操作码。`52`是操作码OP_2，表示数值2。`9c`是操作码OP_NUMEQUAL，用来判断两个数值是否相等。`fifth_elemnet`中首字节代表Tapscript版本号，剩余32字节表示聚合公钥P。
   
   ![Tapscript7](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%847.png?raw=true)

6. nLockTime代表交易允许被打包的最早时间。分四种情况：1）0 表示立即生效。  2）小于 500000000代表块高度，处于该块之前交易锁定。3）大于等于500000000代表Unix时间戳，处于该时刻之前交易锁定。4）交易输入的nSequence字段均为最大值`0xffffffff`时，nLockTime字段无效。

   ![Tapscript8](https://github.com/AAweidai/PictureBed/blob/master/taproot/Tapscript%E4%BA%A4%E6%98%93%E7%BB%93%E6%9E%848.png?raw=true)

## 总结

从交易结构的角度来看，Tapscript沿用隔离见证的交易结构，主要改变在于公钥和签名的更新。如果忽略Tapscirpt版本号和奇偶性等前缀，公钥统一用32字节表示。见证脚本中签名采用Schnorr签名，总长度统一为64字节，省略了ECDSA签名中冗余字节。

