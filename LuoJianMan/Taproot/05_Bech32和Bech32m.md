# Bech32和Bech32m

## Contents

2. [Bech32：隔离见证V0地址格式](#Bech32：隔离见证V0地址格式)
   + [产生Bech32地址](#产生Bech32地址)
3. [Bech32m概览](#Bech32m概览)
   + [Bech32m](#Bech32m)
4. [总结](#总结)

## Bech32：隔离见证V0地址格式

`隔离见证`（SegWit）是 2017 年 8 月发生的的比特币协议更新，SegWit 有效地将签名数据或见证数据与比特币交易分开或隔离，这样做使比特币区块能够容纳更多交易，从而提高比特币网络的可扩展性。此外，将签名数据与比特币交易分离为第二层扩展解决方案（Layer 2）提供了机会，如闪电网络。 

`Bech32` 是一个与 SegWit 完全兼容的比特币地址。 许多人将 Bech32 地址称为 `bc1` 地址，因为它们的地址字符串总是以`bc1`开头。Bech32作为比特币改进提案（BIP-173）的一部分，在应用之前，比特币使用base58地址和截断的双SHA256校验和。

但是，这种实现有一些缺点，因为它使用混合大小写字母，导致base58 地址很难输入，并且在发送比特币时很容易出现输入错误。 想象一下，尝试正确输入 `1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2` 而无需复制和粘贴， 会发现是一件很困难的事。此外，base58 地址用二维码来表示需要大量存储空间，并且解码也相对复杂且缓慢。将所有这些问题与双SHA256 校验和所需的时间较长结合起来，就会开始明白为什么需要切换。

### 产生Bech32地址

Bech32 地址是从公钥创建的，如下所示：

1. 对一个压缩的公钥（0x02 或 0x03 后跟 32 字节的 X 坐标值）： `0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`执行执行 SHA-256 哈希运算，得到`0f715baf5d4c2ed329785cef29e562f73488c8a2bb9dbc5700b361d54b9b0554`

2. 再对哈希结果进行RIPEMD-160散列：`751e76e8199196d454941c45d1b3a323f1433bd6`

3. 第 2 步的结果是一个 8 位无符号整数数组（基数 2^8=256），Bech32 编码将其转换为一个 5 位无符号整数数组（基数 2^5=32），因此我们“挤压”要获取的字节，如调用如下函数`convertbits('751e76e8199196d454941c45d1b3a323f1433bd6', 8, 5)`。

    ```python
    def convertbits(data, frombits, tobits, pad=True):
        """General power-of-2 base conversion."""
        acc = 0
        bits = 0
        ret = []
        maxv = (1 << tobits) - 1
        max_acc = (1 << (frombits + tobits - 1)) - 1
        for value in data:
            if value < 0 or (value >> frombits):
                return None
            acc = ((acc << frombits) | value) & max_acc
            bits += frombits
            while bits >= tobits:
                bits -= tobits
                ret.append((acc >> bits) & maxv)
        if pad:
            if bits:
                ret.append((acc << (tobits - bits)) & maxv)
        elif bits >= frombits or ((acc << (tobits - bits)) & maxv):
            return None
        return ret
    ```

4. 得到5位无符号整数数组`01110 10100 01111 00111 01101 11010 00000 11001 10010 00110 01011 01101 01000 10101 00100 10100 00011 10001 00010 11101 00011 01100 11101 00011 00100 01111 11000 10100 00110 01110 11110 10110`，十六进制表示`0e140f070d1a001912060b0d081504140311021d030c1d03040f1814060e1e16`

5. 对结果添加隔离见证版本号（Segwitness version）字节（当前版本为0），得到`000e140f070d1a001912060b0d081504140311021d030c1d03040f1814060e1e16`

6. 使用上一步结果得到的数据和 HRP（主网为 bc，测试网为 tb），计算校验和`0c0709110b15`，附加到上一步结果，得到`000e140f070d1a001912060b0d081504140311021d030c1d03040f1814060e1e160c0709110b15`

7. 将每个值映射到 Bech32的字符集（qpzry9x8gf2tvdw0s3jn54khce6mua7l，如00 -> q, 0e -> w,...）得到qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4

    + ![bech32-charset](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1628674702208-1628674702206.png)

8. 最终得到一个 Bech32_encoded地址，`bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4`

其中，bech32地址有四个组成部分：

1. 人类可读部分 `HRP`
2. 数字`1`作为分隔符
3. base32编码后的字符串，包含有效的地址数据
4. 使用 BCH 算法编码生成的的 6 字符 base-32 编码纠错码

支持 Bech32可以减小发送和接收比特币时出错的概率， 因为 Bech32 地址完全小写，而不是混合大小写，用户可以更轻松地共享并输入它们。如果用户在输入地址时确实出错，Bech32 地址还可以让您别哪些字符可能不正确。

## Bech32m概览

BIP173 定义了 `Bech32` 的通用校验和 base 32 编码格式，它用于版本 0（P2WPKH 和 P2WSH）的隔离见证输出和其他应用程序。但Bech32 有一个意想不到的缺陷：只要最后一个字符是 `p`，在它前面插入或删除任意数量的 `q` 字符，都不会使校验和无效。由于存在对两个特定长度的限制，这不会影响SegWit 版本 0 地址的现有使用，但可能会影响使用 Bech32 编码的未来使用，其他应用程序若使用bech32地址也可能会受到影响。

`Bech32m` 是Bech32的一个优化变体，可缓解这个插入缺陷和相关的问题，并修改了 BIP173 以将 Bech32m 用于版本 1 及更高版本的原生隔离见证输出，而Bech32 仍然用于版本 为0 的隔离见证输出。

### Bech32m

Bech32m 修改了 Bech32 规范的校验和，用 `0x2bc830a3` 替换了最后异或到校验和中的常数 `1`。

```python
BECH32M_CONST = 0x2bc830a3

def bech32m_polymod(values):
  GEN = [0x3b6a57b2, 0x26508e6d, 0x1ea119fa, 0x3d4233dd, 0x2a1462b3]
  chk = 1
  for v in values:
    b = (chk >> 25)
    chk = (chk & 0x1ffffff) << 5 ^ v
    for i in range(5):
      chk ^= GEN[i] if ((b >> i) & 1) else 0
      # k(x) = {29}x^5 + {22}x^4 + {20}x^3 + {21}x^2 + {29}x + {18}
      # {2}k(x) = {19}x^5 +  {5}x^4 +     x^3 +  {3}x^2 + {19}x + {13}
      # {4}k(x) = {15}x^5 + {10}x^4 +  {2}x^3 +  {6}x^2 + {15}x + {26}
      # {8}k(x) = {30}x^5 + {20}x^4 +  {4}x^3 + {12}x^2 + {30}x + {29}
      # {16}k(x) = {21}x^5 +     x^4 +  {8}x^3 + {24}x^2 + {21}x + {19}
  return chk

def bech32m_hrp_expand(s):
  return [ord(x) >> 5 for x in s] + [0] + [ord(x) & 31 for x in s]

def bech32m_verify_checksum(hrp, data):
  return bech32m_polymod(bech32m_hrp_expand(hrp) + data) == BECH32M_CONST

def bech32m_create_checksum(hrp, data):
  values = bech32m_hrp_expand(hrp) + data
  polymod = bech32m_polymod(values + [0,0,0,0,0,0]) ^ BECH32M_CONST
  return [(polymod >> 5 * (5 - i)) & 31 for i in range(6)]
```

Bech32 的所有其他方面保持不变，包括其人类可读部分（HRP)，可以使用以下代码编写同时解码 Bech32 和 Bech32m 的组合函数。它返回 None 表示失败，或 BECH32 / BECH32M 枚举值之一以指示根据相应标准进行解码。

```python
class Encoding(Enum):
    BECH32 = 1
    BECH32M = 2

def bech32_bech32m_verify_checksum(hrp, data):
    check = bech32_polymod(bech32_hrp_expand(hrp) + data)
    if check == 1:
        return Encoding.BECH32
    elif check == BECH32M_CONST:
        return Encoding.BECH32M
    else:
        return None
```

SegWit 版本 0 输出（特别是 P2WPKH 和 P2WSH 地址）将继续使用 BIP173 中指定的 Bech32，隔离见证输出版本 1 到 16 的地址将使用 Bech32m。同样，编码的所有其他方面保持不变，包括`bc`（HRP）。要解码比特币网络地址，客户端软件应该使用 Bech32 和 Bech32m 解码器进行解码，或者使用同时支持两者的解码器。在这两种情况下，地址解码器都必须验证编码是否与解码的隔离见证版本（版本 0 的为 Bech32，其他版本的为 Bech32m）相匹配。如进行以下检查。

```python
def decode(hrp, addr):
    hrpgot, data, spec = bech32_decode(addr)
    if hrpgot != hrp:
        return (None, None)
    decoded = convertbits(data[1:], 5, 8, False)
    # Witness programs are between 2 and 40 bytes in length.
    if decoded is None or len(decoded) < 2 or len(decoded) > 40:
        return (None, None)
    # Witness versions are in range 0..16.
    if data[0] > 16:
        return (None, None)
    # Witness v0 programs must be exactly length 20 or 32.
    if data[0] == 0 and len(decoded) != 20 and len(decoded) != 32:
        return (None, None)
    # Witness v0 uses Bech32; v1 through v16 use Bech32m.
    if data[0] == 0 and spec != Encoding.BECH32 or data[0] != 0 and spec != Encoding.BECH32M:
        return (None, None)
    # Success.
    return (data[0], decoded)
```

## 总结

Bech32m 通过更改Bech32编码方案中使用的常量来消除此漏洞，因此更安全。Bech32m 被提议作为 SegWit 版本 1（即Taproot）地址的编码方案，它将由 Taproot 升级引入。在引入后，为隔离见证输出生成地址时，如果其见证版本为 0，则使用 Bech32 对其进行编码；如果其见证版本为 1 或更高，则使用 Bech32m 对其进行编码。

