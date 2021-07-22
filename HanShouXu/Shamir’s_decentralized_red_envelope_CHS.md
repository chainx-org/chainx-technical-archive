# Shamir’s 门限红包方案

本文将回顾去中心化红包方案和提出新的基于Shamir's门限的红包分发方案，但基于Shamir's门限的方式事实上是为了保证红包参与者的基数和保证系统一定的容错性，在整个红包运作流程中会增加负担，可以结合相应应用场景选择

## 一、去中心化红包方案简述

在原有的去中心化红包方案中我们使用了randao的随机数生成框架，基于多人参与的随机数共同生成保证了随机数的真实随机和不可操纵。
在此基础上，我们加入了VRF的可验证随机，保证了参与者可对自己参与性的证明，相比原有集中式红包分发更加公平。
去中心化红包的流程如下：

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random.png)

    1、红包发送者Sender发送红包给智能合约，包含红包金额和总手续费，智能合约以此告知可参与者参与领取红包
    
    2、决定领取的参与者将个人随机散列值和抵押金发送给智能合约并生成可验证零知识证明由合约验证（默认6个区块时间）
    
    3、合约验证散列值和随机数的有效性，无效参与则扣除抵押金并取消领取资格，有效参与者共同参与红包随机生成
    
    4、合约分发红包给有效参与者并返还抵押金

详细流程可以参考去[中心化红包方案](https://github.com/chainx-org/chainx-technical-archive/blob/main/The%20decentralized%20red%20envelope/%E5%8E%BB%E4%B8%AD%E5%BF%83%E5%8C%96%E7%BA%A2%E5%8C%85.pdf)

## 二、Shamir's门限秘密共享

(t，n)门限秘密共享方案，该方案是Shamir和Blakley在1979年各自独立地提出，(t，n)门限方案是基于(t，n)门限访问结构上的秘密共享方案，而(t，n)门限访问结构包括所有t个或t个以上的参与者子集所构成的集合。

简单来说，一个秘密数据s被拆分为n个份额，掌握t份或者更多的秘密s份额可以使s非常容易恢复计算，掌握t-1或者更少的秘密s份额使恢复s在理论上不可行。这种门限的实现基于拉格朗日插值法：

### 拉格朗日插值法

在数值分析中，拉格朗日插值法是以法国18世纪数学家约瑟夫·拉格朗日命名的一种多项式插值方法。我们可以这样理解，已知下面这几个点，我们需要找到一根穿过它们的曲线：

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_1.png)

我们可以合理的假设，这根曲线是一个二次多项式：y = a0 + a1x + a2x^2，而我们知道已有的三个点，所以可以通过方程组解出这个二次多项式，而拉格朗日认为可以通过三根二次曲线的相加来达到目标：

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_2.png)
![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_3.png)
![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_4.png)

而这三根曲线就是拉格朗日所需要的，因为y1f1(x)可以保证在x1点处取值为y1，其余两点取值为0，同理y2f2(x)可以保证在x2点处取值为y2，其余两点取值为0，y3f3(x)可以保证在x3点处取值为y3，其余两点取值为0，自然f(x) = y1f1(x) + y2f2(x) + y3f3(x)

### 门限的应用方式

拉格朗日插值是如何应用在门限加密中的？我们可以通过一个例子简单了解：

    已知 t 个二维坐标(x1,y1)......(xt,yt)，x1 到 xt 各不相同
    
    经过 t 个点有且仅有一条 t-1 阶多项式使得对于所有 i = 1...t, f(xi) = yi
    
    假设 s 为秘密, 将 s 分成 n 份，由 Dealer 生成随机系数多项式 f(x) = a0 + a1*x + a2 * x^2 + ..... +a(t-1) * x^(t-1)
    
    使得 a0 为秘密 s, 且制造出n个份额：y1 = f(1), ...... , yn = f(n)
    
    将 n 个份额分发给 n 个参与者，当足够t个份额时，通过插值法解出 f(x)

可以看到，事实上我们可以理解为t个未知数由t个方程可以获得最终的所有参数，而a0就是其中的秘密，f(x)求出时我们令x为0就可以得到最终的秘密a0，这种方式我们可以制造无数超过t个的份额而保证只有在t个份额的情况下可以解出最终f(x)。相关实现代码可以参考[此处资料](https://github.com/randao/randao)

## 三、Shamir's门限红包方案

在我们的原有红包分发算法中如为保证一定基数的红包领取参与者而不是被少部分所完全独占，可以考虑加入Shamir's门限秘密共享算法，基于算法的特性，可以保证至少t个参与者的参与才能收取红包。同时通过秘密共享或门限签名的方式，可以避免随机数生成方案因为一个参与方没有完整执行流程而失败，可以使得去中心化红包系统具备一定的容错性，提高随机数产生的成功率和红包分发效率：

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_5.png)
