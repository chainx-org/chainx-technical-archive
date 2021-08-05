# Musig2基本原理

## Musig2介绍

​	多重签名方案使一组签名者能够在消息上产生联合签名。Musig2是一种多重签名方案，是MuSig 签名方案的一个新变体。MuSig 允许多个签名者从他们各自的私钥中创建一个聚合公钥，然后共同为该公钥创建一个有效签名，通过这个方式创建的聚合公钥与其他公钥是无法区分的。最初的 MuSig 需要三轮签名，但是新的 MuSig2 方案实现了一个简单的只需要两轮的签名协议，而且不需要零知识证明。MuSig2是一种简单且高度实用的两轮多重签名方案，具有优势： i) 在并发签名会话下是安全的，ii) 支持密钥聚合，iii) 输出普通 Schnorr 签名，iv) 只需要两轮通信，v) 具有与普通 Schnorr 签名相似的签名者复杂性。同时，Taproot升级兼容Musig2多签方案。在我们ChainX跨链新方案中，将采用Musig2签名方案来完成聚合签名的过程。

## Musig2原理

![Musig2_1](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_1.png?raw=1)

​	上图来源于原始论文，描述了Musig2签名的基本过程。Musig2签名首先需要参与方各自通过`KeyGen()`生成公私钥并通过`Sign()`生成临时公私钥（即nonce）。然后进行第一轮通信，将临时公钥(即上图中的`out_i`)传递给其他人。拿到其他各方的临时公钥后可以通过`SignAgg()`和`Sign'()`两个函数生成签名碎片`out'_i`。最后进行第二轮通信，将签名碎片`out'_i`传递给其他人，拿到其他各方的签名碎片后，就可以通过`SignAgg'()`和`Sign''()`生成最终的签名`σ`。图中`KeyAgg()`表示公钥聚合函数，`Ver`表示Musig2验签函数。Musig2聚合签名最终形式和Schnorr普通签名的形式一致，可以表示为（R, s）。下文展示了rust版本Musig2的部分实现。

### 签名

1. 生成密钥: 各方生成随机私钥并计算相应公钥

   ![Musig2_2](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_2.png?raw=1)

2. 生成nonce：各方生成临时公私钥。在函数`create_vec_from_private_key()`中，Musig2为了两轮通信完成聚合签名，会产生`Nv`对临时公私钥。`sign()`函数的返回值`msg`包含了所有的临时公钥，可以通过p2p网络传递给其他人。下图代码中通过`party1_received_msg_round_1`和`party2_received_msg_round_1`模拟了该过程。

   ![Musig2_3](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_3.png?raw=1)

3. 生成签名碎片：下面代码中通过`sign_prime()`函数完成了论文中`SignAgg()`和`Sign'()`两个函数的功能。参数中`message`表示待签名的消息，`pks`表示各个参与方所有公钥的有序列表，`party_index`代表当前参与方的公钥在`pks`中的索引。`sign_prime()`函数主要是利用私钥`self.keypair`和临时公钥`msg_vec`聚合成的`c, R, b_coefficients`来生成签名碎片`s_i`。这里的`c`表示`message`进行hash生成的消息摘要，`R`是Musig2聚合签名最终结果的第一项，`b_coefficients`是生成签名碎片的系数。通过p2p网络将签名碎片`s_i`传递给其他人。下图代码中通过`party1_received_msg_round_2`和`party2_received_msg_round_2`模拟了该过程。

   ![Musig2_4](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_4.png?raw=1)

4. 聚合签名碎片：收集到其他各方的签名碎片后，通过`sign_double_prime()`生成Musig2聚合签名的第二项。该函数主要是对签名碎片进行累加操作。`s_total_1`和`s_total_2`表示参与各方各自生成的聚合签名碎片，正常情况下，两者是一致的。

   ![Musig2_5](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_5.png?raw=1)

### 验签

![Musig2_6](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig2_6.png?raw=1)

​	上图通过`verify()`函数展示了Musig2的验签过程。参数中`r_x, signature`对应Musig2聚合签名的（R，s），`X_tilde`代表聚合公钥，`c`和前文一致，表示`message`进行hash生成的消息摘要。

## 总结

​	总的来看，Musig2只需通过两轮通信即可完成聚合签名的过程。相比于Musig，主要差别在于nonce的生成过程。其中Musig2每一个参与方生成了多对临时公私钥，而Musig仅仅生成一对临时公私钥。这也是Musig2仅仅需要两轮通信的根本原因。同时，Musig2是利用Schnoor签名基本原理形成的多签方案，可以很好的兼容Taproot升级并应用于我们的ChainX跨链升级新方案中。