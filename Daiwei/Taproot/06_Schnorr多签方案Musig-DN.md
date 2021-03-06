# MuSig-DN

## 简介

​	多重签名方案允许一组签名者（每个人都有自己的密钥对）协作计算公共消息的签名。MuSig（又叫MuSig1），MuSig2，MuSig-DN一脉相承，都描述了一种多签解决方案。MuSig需要进行三轮通信完成一次多签。MuSig2在MuSig基础上消除了一轮通信，并允许使用类似于我们目前基于脚本的多重签名过程。 但MuSig2存储额外的数据，并需要非常小心确保签名软件或硬件不会被欺骗而在不知不觉中重复部分签名会话。MuSig-DN通过零知识证明和 Purify（一种有效的确定性随机数推导函数）的组合，保护 用户免受由于不良随机数生成器和虚拟机重置攻击引起的密钥泄露攻击。同时MuSig-DN不易受到重复会话攻击。

## 随机性和确定性

​	每个 Schnorr 或 ECDSA 签名都需要一个nonce，即签名者生成的秘密随机数。这些签名方案的安全性在很大程度上依赖于这种随机数的均匀随机生成，即使随机性中的轻微偏差也可能使攻击者从签名中提取密钥成为可能。糟糕随机性的一个简单而极端的例子，就是为不同的消息重用随机数。如果使用相同密钥和完全相同的随机数对两条不同消息进行签名，则可以立即从这两个签名中计算出该密钥。
​	为了避免签名过程中的这种脆弱性，通常使用确定性随机数，例如，使用哈希函数作为随机数 `SHA256(sk, m)` ，其中 sk 是签名者的密钥，m是要签名的消息。由于攻击者不知道 sk，因此产生的随机数仍然是不可预测的，即使它实际上是从确定性计算而不是真正的随机性得出的。由于 m 是散列函数输入的一部分，不同消息的签名将使用不同的随机数，这排除了不同消息的随机数重用。确定性随机数还可以防止回滚攻击，在这种攻击中，攻击者试图通过重置签名算法在两个不同的消息上获得具有相同随机数的签名。
​	由于这些优点，在生成签名时使用确定性随机数是轻而易举的，并且实际上所有签名的严肃实现都依赖于某种方法来确定性地导出随机数。然而，当涉及到多重签名时，如果天真地使用确定性随机数并不能防止暴露密钥，令人惊讶的是，情况恰恰相反具有确定性随机数的 MuSig 实现更容易暴露密钥。攻击的工作原理如下：恶意签名者与受害者为同一消息签名进行两次签名会话。由于消息在两个会话中是相同的，因此受害者确定性地生成完全相同的随机数。但是 MuSig 中没有任何内容会强制另一个恶意签名者确定性地生成他的随机数。恶意签名者可以使用任意随机数。因此，两个签名会话中的“聚合随机数”（根据所有签名者的随机数计算）将不同。最终拥有不同的聚合 nonce 与拥有不同的消息具有相同的效果，并且攻击者可以轻松地从这两次签名会话中提取受害者的密钥。

## Musig-DN

​	MuSig-DN是克服上述困难并在 MuSig 中安全使用确定性随机数的多签方案。 MuSig-DN 背后的主要思想是使用零知识证明来强制所有签名者确定性地生成他们的随机数。每个签名者从一个额外的秘密密钥（这里称为 nonce 密钥）、消息和所有签名者的公钥中派生出一个确定性的随机数。然后，签名者在第一轮 MuSig-DN 中传达他们的公共随机数以及证明他们的公共随机数已正确导出的非交互式零知识证明。所有其他签名者都可以使用与签名者的 nonce 密钥关联的附加公钥（这里称为主机密钥）来检查这些证明。如果任何签名者试图通过在不同的签名会话中发送两个不同的随机数来作弊，其他签名者将检测到它（因为两个随机数中的至少一个将具有无效的零知识证明）并简单地中止协议。
​	除此之外，MuSig-DN 的另一个主要贡献是 Purify，这是一种用于导出随机数的特定函数。与 SHA256 之类的散列函数相反，它是派生确定性随机数的典型选择。Purify 依靠椭圆曲线来提取看似随机的随机数，并 可以有效地与零知识证明框架一起使用。
​	MuSig-DN 的一个非常好的附加属性是：和MuSig2类似，签名会话只需要两轮通信即可完成。

## Musig对比

![Musig-DN](https://github.com/AAweidai/PictureBed/blob/master/taproot/Musig-DN.png?raw=1)

​	上图对Musig系列的多签方案进行了简单的对比。从上图来看，没有理由比 MuSig2 更喜欢 MuSig1。相比于MuSig2，MuSig-DN 的优势在于它支持确定性随机数，这避免了在签名会话和轮次之间保持状态的需要，减少重复会话攻击。但是MuSig-DN 的劣势在于不支持非交互式签名，同时实现复杂度和计算复杂度特别高。

# 结论

​	总的来看，MuSig-DN是通过零知识证明和 Purify的组合，实现的确定性随机数多签方案。MuSig-DN避免了签名会话和轮次之间保持状态的需要，从而有效减少重复会话攻击。