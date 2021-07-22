# understand Schnorr and Musig

## Content

1. [Schnorr Signature](#Schnorr%20Signature)
	+ [encryption methods in Elliptic Curves](#encryption%20methods%20in%20Elliptic%20Curves)
	+ [implementation principle of Schnorr signature](#implementation%20principle%20of%20Schnorr%20signature)
	+ [sign the message](#sign%20the%20message)
	+ [verify the signature](#verify%20the%20signature)
	+ [why nonces are necessary](#why%20nonces%20are%20necessary)
2. [Implementation of Musig with Schnorr signature](#Implementation%20of%20Musig%20with%20Schnorr%20signature)
	+ [aggregate pubkeys](#aggregate%20pubkeys)
	+ [key cancellation attack](#key%20cancellation%20attack)
	+ [aggregate nonces](#aggregate%20nonces)
	+ [aggregate signatures](#aggregate%20signatures)
	+ [the complete process of Musig three-round communication](#the%20complete%20process%20of%20Musig%20three-round%20communication)

## Schnorr Signature

### encryption methods in Elliptic Curves

​		Before you learn about Schnorr signatures, the first thing to do is to create public and private key pairs based on elliptic curves. Points on elliptic curves can perform some algebraic operations, involving the concepts of scalar and point. Scalars are positive integers, represented by lowercase letters (for example, k), a point on a curve, represented by uppercase letters (such as A), or a pair of coordinates (for example, (x, y)). Scalars and points support the operation of the following figure.

![An overview of all operations of scalars and points over elliptic curves.](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626838499243-1626838499234.png)

​		We can see from the graph that multiplication and division can be performed between scalars and points, but multiplication and division can not be satisfied between points. To expand on this point, if there is a point GMague G that adds itself many times to `R = G + G + G = 3 * G`, G and R can be regarded as points before and after encryption, and 3 is a scalar (actually a very large number). If 3 and G are known, then `R = 3 * G` can be easily calculated, but if you only know the encrypted point R and the pre-encrypted point G, I can't get a scalar k equals 3 from`R / G`, because the point cannot satisfy the operation. The above situation is similar to the generation of public and private key pairs. The scalar k is regarded as the private key, and the ordinate of the encrypted point is regarded as the public key. The only feasible way to reverse crack the private key k is to guess how much the scalar k is, `k * G = R`, and enumerate k to get the answer. In Bitcoin, the public and private key pairs are generated and signed through the SECP256K1 curve. The value of the scalar is an integer value between 0 and 2 to the 256th power, which is equal to the number of atoms in the whole universe, so there are endless possibilities, thus ensuring the security of the generated public and private key pairs.

​		An elliptic curve is a set of elliptic curves defined by ![formula-a](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626849064158-1626849064158.png) and satisfying ![formula-b](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626849094992-1626849094991.png) 's set of points, where different parameter values correspond to different elliptic curves. For the value of each X-axis, corresponding to the values `y` and`-y` of the Y axes, only one of the values of these two ordinates satisfies the quadratic residue of the modulus of `SECP256K1_FIELD_SIZE`, which can be determined by `jacobi_symbol`, and this valid y value is the public key. Therefore, from the private key d, a point `P = k* G` can be generated without ambiguity, and its valid y-axis value can be obtained as the public key.

![curve](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626849644267-1626849644265.png)

​	The above asymmetry is the premise of the discussion of Schnorr signature.

### implementation principle of Schnorr signature

![Schnorr Signatures And Verification](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626847754743-1626847754740.png)

To sign a message m with Schnorr, you first need to define several variables:
- G: Elliptic curve.
- m: the data to be signed, usually a 32-byte hash.
- d, P: private key d and public key P held by the user, where `P = d * G`.
- H (): hash function.
- Other: `H (m | R | P)` can be understood as stitching the m, R, P fields together and then hashing.
-another: X (R) represents the horizontal coordinate value of point R

####  sign the message

The following methods are used to create Schnorr signatures:

+ create a random number k, and create a random point R from k, where `R = k* G`.
+ calculate scalar `s = k + H (x (R) | P | m) * d`, where P is the public key, d is the private key and m is the message.
+ the final Schnorr signature `S = (x(R), s)` is a pair of values, which is composed of 32 bytes of x(R) and 32 bytes of s, and the final length is 64 bits.

![schnorr-sign](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626854727113-1626854727110.png)

The result of Schnorr signature is as follows:

```json
message = ab530a13e45914982b79f9b7e3fba994cfd1f3fb22f71cea1afbf02b460c6d1d
user privkey = 40811356790294382983149959962124660206130438370548668724053064036307538116679
user pubkey = 0374248e7fdb13546ac94d961365aff6d352c413dd79b2056a2bb60f2971e79fc6

nonce: 55221941004635623701832462325081428209726520449893387818054719258463597914212
nonce point: 039c5530a2e78faa9a87d12ea48f201cec4462e21237ee6e682c935a28a44b826d

R: 039c5530a2e78faa9a87d12ea48f201cec4462e21237ee6e682c935a28a44b826d
x(R) = 9c5530a2e78faa9a87d12ea48f201cec4462e21237ee6e682c935a28a44b826d
s: 2265af70948f333f49fd1f4d38b4791cb8682f05d9719e5f482ef99b7dff790d

Signature: 9c5530a2e78faa9a87d12ea48f201cec4462e21237ee6e682c935a28a44b826d2265af70948f333f49fd1f4d38b4791cb8682f05d9719e5f482ef99b7dff790d
```

#### verify the signature

​		Anyone can still verify the Schnorr signature without knowing the random number k and the private key d. What the verifier knows is G-elliptic curve, H ()-hash function, m-message to be signed, P-public key, x (R) and s-Schnorr signature. Verification equation: `S = R + H (x (R) | P | m) * P`. If it is true, the signature is valid.

​		To deduce this process, it contains an extremely important theory: there is no division between the points in the elliptic curve and the points in the elliptic curve.

1. `s = k + H (m | R | P) d`, if both sides of the equation are multiplied by the elliptic curve G, there are:

2. `sroomG = karmG + H (m | R | P) * d* G`, and because `R = kumbg, P = x* G`, there are:

3. `sSecretG = R + H (m | R | P) * P`, the elliptic curve cannot be divided, so if the equation in step 3 cannot be deduced back to step 1, the value of k and the private key of x will not be exposed. At the same time, the equation verification is completed.

#### why nonces are necessary

​		If we only sign a message m, then `h = H (P | m) `, we get the scalar `s = h* d`. Since there is no random number, we get a 32-bit Schnorr signature `S = s = h*d`. For this result, calculate whether `S = H(P | m) * P` is valid, and we can verify whether the signature is valid as usual.

​		But in this case, anyone can calculate what the private key d is, because s is a scalar, so `d = s / h` can be easily calculated; if you add a random number `k`, you must solve `d = (s-k) / h`, but the random number `k` is unknown, so this calculation is not feasible, which explains why random numbers are introduced.

## Implementation of Musig with Schnorr signature

​		Because the points on the elliptic curve can be linearly accumulated, the random numbers and public keys of multiple users can be linearly accumulated to generate aggregate random numbers and aggregate public keys. Accordingly, the concept of an "aggregation account" is generated. Using this "aggregation account" to complete the above Schnorr signature process is basically the same as that of ordinary accounts for Schnorr signature. Therefore, after the introduction of Schnorr signature into the Bitcoin main network, there is no difference between aggregate signatures and ordinary transactions in terms of transaction results, which protects the privacy of users who participate in aggregate signatures to a great extent.

​		The aggregate Musig scheme consists of two parts, the aggregate public key and the aggregate random number, and the signature. The offline process of aggregating public keys does not require communication, and the process of aggregating random numbers and signatures requires three rounds of communication (under the Musig2 scheme, it is improved to two rounds of communication, see the previous article).

![musig-overview](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626919916096-image-20210721163826794.png)

### aggregate pubkeys

​		The aggregate public key is to linearly add the public key of each participant. This process is easy to accomplish, that is, `P_agg = P_0 + P_1 + ... + P_n`, but key cancellation attacks need to be considered.

#### Key cancellation attack

​		We can imagine a scenario in which Alice and Bob want to generate 2-2 aggregate signatures, and in order to achieve the purpose of aggregation, they need to exchange the public key and random number needed for the above Schnorr signature. But if Bob knows the public key `P_a` and random number `R_a` of Alice beforehand, then Bob can deceive Alice by sending `P'_b = P_b - P_a` and `R'_f = R_b - R_a` to Alice when Alice calculates the signature. The following will happen.

![key cancellation](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626919973621-image-20210721171442302.png)

​		As can be seen from the results, the `Ra` and `Pa` of Alice are offset in the final signature result, which means that Bob can control the assets of the aggregate signature alone, which is obviously dangerous for Alice.

​		Of course, we can challenge Bob to let Bob prove that he owns the private key scalar of the declared public key P and random number R, but this requires one more round of communication, so we can adjust the participant's public key by adding a challenge factor to counteract the key cancellation attack as follows:

​		The Pubkey of each participant is adjusted by the challenge factor, which is unique to each participant, and all challenge factors are generated based on the participants' aggregate public key, which ensures that no individual participant (or group of participants) can create a public key that offsets the public keys of other participants. As long as everyone has access to the public keys of all participants (through offline communication, coordination, and so on), challenge factors and aggregate public keys can be calculated locally without the need for an additional round of communication.

![image-20210721172330638](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626920000310-image-20210721172330638.png)

### aggregate nonces

​		Each participant has to generate his own random number `k_i` and random number point `R_i` (I = 0,1..., indicating the participant's order). Then, participants exchange these random number points `R_i` and linearly add all random number points `R_i` to aggregate random number point ``R_agg`. This process is very similar to the above aggregate public key, but it cannot be verified securely in the same way as the aggregate public key, because the participant's public key is unchanged, even if multiple signatures are carried out multiple times and offline coordination is carried out. The final aggregate public key is still the same. However, the random number is different, and the random number in each signature process is different. In order to avoid cheating in the exchange process, it is inevitable to add a round of communication. In this round of communication, participants exchange hash promises of random number points `R_i`.

​		Each participant will exchange the random number `R_i` after receiving all the promises of the random number `R_i`, and verify that the random number `R_i` matches the given hash commitment, if.

​		If it is confirmed that it is correct, it will be calculated: the conforming module `SECP256k1 _ FIELD_SIZE` is the valid aggregate random number `R_agg` of the quadratic remainder.

![agg-nonces](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626920030312-image-20210721180219390.png)

### aggregate signatures

​		After each participant calculates the aggregate public key and aggregate random number, the signature calculation can be performed. Each participant has a private key ``di` and a random number `k_i`. The result obtained by calculating the scalar `s_i = k_i + H(x(R_agg) | P_agg | m) * d_i` is a partial signature. Finally, the signature result is exchanged, and the partial signature of each participant is linearly added to generate the final aggregate signature.

![agg-sigs](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626920048629-image-20210721180434124.png)

### the complete process of Musig three-round communication

![musig-1](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626863264267-1626863264264.png)

![musig-2](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626921006922-image-20210721182830058.png)

![musig-3](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1626863344878-1626863344875.png)

The results are as follows:

```json
Individual public key is:
	026c5d5e73124f3c821c0985df787e11b3d018a86add577fa8661613a0d49dde59, 
	03f771877964fa2ce401d87bc2558a0df1e6921acef99389f059712b32cfda35fd, 
	03f039fdcdb728efbbddf4ee452419a988497debb7bd1b42644c5fa66e9af8c8b6.

Aggregated Public Key is 02eeeea7d79f3ecde08d2a3c59f40eb3adcac9defb77d3b92053e5df95165139cd

Tweaked pubkey1 is 02066483b0841dba5821ee719178fb878102262cf505975e0a2d18e48c84b8362e
Tweaked pubkey2 is 0376e2769bcdc42a20e18cfa83125435c0cd1348a7849c6feebeb9394776cdcea6
Tweaked pubkey3 is 03bbc4110f3b28023f6ffefd29de76fe915029ce693f8fc3a1c327bdeb73940840
Individual nonce scalars:
	115792089237316195423570985008687907852837564279074904382605163141518161494236, 
	115792089237316195423570985008687907852837564279074904382605163141518161494115, 
	115792089237316195423570985008687907852837564279074904382605163141518161494004.

Aggregate nonce point: 03f90c3416d74049bf27b5563067c58401ff466e4bb04e1fa4d51ae4c93b4a8316

Partial signatures:
	65632340538892058604021005685526525791383877758802541334726676556343496273695
	49071424722101348040708779394444974474178306070453713897027096230295229616215
	40052638870461774859002446004708555184684360378078494722527781424448839071327

Aggregate signature:
	f90c3416d74049bf27b5563067c58401ff466e4bb04e1fa4d51ae4c93b4a83165625054ca06a0e7a76ecca379955370d56fa014fc1c0e62313dd4ed246b23494

Success!
```

