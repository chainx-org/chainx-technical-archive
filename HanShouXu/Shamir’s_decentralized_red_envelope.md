# Shamir’s threshold Red Envelope Scheme

This article will review the decentralized red envelope scheme and propose a new red envelope distribution scheme based on Shamir's threshold, but the method based on Shamir's threshold is in fact to ensure the cardinal number of red envelope participants and ensure a certain fault tolerance of the system. However, it will increase the burden in the entire operation process, which can be combined with the corresponding application scenarios.

## 一、Brief description of the decentralized red envelope solution

In the original decentralized red envelope scheme, we used Randao's random number generation framework, based on the joint generation of random numbers with multiple participants to ensure that random numbers are truly random and unmanipable.
On this basis, we have added the verifiable randomness of VRF to ensure that participants can prove their participation, which is more fair than the original centralized red envelope distribution.
The process of decentralized red envelopes is as follows:

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random.png)

1. The red envelope sender sends a red envelope to the smart contract, including the amount of the red envelope and the total handling fee, and the smart contract informs the participants to participate in the red envelope collection
    
2. Participants who decide to receive will send their personal random hash value and collateral to the smart contract and generate verifiable zero-knowledge proof to be verified by the contract (default 6 block time)
    
3. The contract verifies the validity of the hash value and the random number. Invalid participation will deduct the deposit and cancel the eligibility. Effective participants will participate in the random generation of red envelopes.
    
4. The contract distributes red envelopes to valid participants and returns the deposit

The detailed process can refer to the [decentralized red envelope solution](https://github.com/chainx-org/chainx-technical-archive/blob/main/The%20decentralized%20red%20envelope/The%20decentralized%20red%20envelope.pdf)

## 二、Shamir's threshold secret sharing

The (t, n) threshold secret sharing scheme was proposed by Shamir and Blakley independently in 1979. The (t, n) threshold scheme is a secret sharing scheme based on the (t, n) threshold access structure. The (t, n) threshold access structure includes all t or more participant subsets.

To put it simply, a secret data s is split into n shares. Mastering t shares or more secret s shares can make it very easy to recover calculations, but mastering t-1 or less secret s shares cannot restore s. The realization of this threshold is based on Lagrangian interpolation:

### Lagrange interpolation

In numerical analysis, Lagrange interpolation is a polynomial interpolation method named by French mathematician Joseph Lagrange in the 18th century. We can understand it this way. Given the following points, we need to find a curve that passes through them:

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_1.png)

We can reasonably assume that this curve is a quadratic polynomial：y = a0 + a1x + a2x^2，and we know the existing three points, so we can solve this quadratic polynomial through the system of equations, but Lagrange believes that the goal can be achieved through the addition of three quadratic curves：

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_2.png)
![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_3.png)
![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_4.png)

These three curves are what Lagrangian needs, because y1f1(x) can be guaranteed to take the value y1 at the point x1, and the other two points take the value 0. Similarly, y2f2(x) can take the value y2 at the point x2. , The other two points take the value 0, y3f3(x) can take the value y3 at the x3 point, and the other two points take the value 0, naturally f(x) = y1f1(x) + y2f2(x) + y3f3(x)

### How the threshold is applied

How is Lagrangian interpolation used in threshold encryption? We can briefly understand through an example:

1、Given t two-dimensional coordinates (x1,y1)......(xt,yt), x1 to xt are different
    
2、There is one and only one polynomial of degree t-1 through t points such that for all i = 1...t, f(xi) = yi
    
3、Suppose s is a secret, divide s into n parts, and the Dealer generates a random coefficient polynomial f(x) = a0 + a1*x + a2 * x^2 + ..... +a(t-1) * x^( t-1)
    
4、Let a0 be the secret s, and make n shares: y1 = f(1), ......, yn = f(n)
    
5、Distribute n shares to n participants, when there are enough t shares, solve f(x) by interpolation

It can be seen that, in fact, we can understand that there are t unknowns, we can solve all the parameters through t equations, and a0 is the secret among them. After f(x) is calculated, we can get the result by setting x to 0. . In this way, we can create countless more than t shares and guarantee that the final f(x) can be solved only in the case of t shares.Related implementation code can refer to [data](https://github.com/randao/randao)
## 三、Shamir's threshold Red Envelope Scheme

In our original red envelope distribution algorithm, if you want to ensure that a certain number of red envelope recipients are not completely monopolized by a small number of participants, you can consider adding Shamir's threshold secret sharing algorithm.

![image](https://github.com/Han-sx/Random-number-generator/blob/main/image/random_5.png)

Based on the characteristics of the algorithm, it can be guaranteed that at least t participants can participate in order to receive red envelopes. At the same time, by means of secret sharing or threshold signature, the random number generation scheme can be prevented from failing because a participant does not have a complete execution process, which can make the decentralized red envelope system have a certain degree of fault tolerance, and improve the success rate of random number generation and the efficiency of red envelope distribution.
