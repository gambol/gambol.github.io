---
layout:     post
title:      "BTC地址生成过程"
subtitle:   " \"用代码来生成一个地址\""
date:       2018-04-30 12:00:00
author:     "gambol"
header-img: "img/post-bg-2015.jpg"
tags:
    - 数字货币
    - BTC
---

## 概述
比特币建立在加密学基础上, 中本聪利用ECDSA(椭圆加密算法)来产生比特币的公钥和私钥.
私钥 -> 公钥 -> 比特币地址

地址其实是另一种格式的公钥.

### HEX
在系统里任何计算时, 永远是byte[]. 而我们看到的是都是16进制(HEX之后)的结果.

譬如,byte数组是`{20 30}`
HEX之后的结果是`141e`

因此, 如果要输出时, 记得先 `HEX.encode(byte[])`
处理输入时, 需要先 `HEX.decode(String)`


## 具体生成过程
1. 随机选择一个32字节的数, 在1 ~ 0xFFFF FFFF FFFF FFFF FFFF FFFF FFFF FFFE BAAE DCE6 AF48 A03B BFD2 5E8C D036 4141之间, 作为私钥
`18e14a7b6a307f426a94f8114701e7c8e774e7f9a47e2c2035db29a206321725`

2. 使用ECDSA计算私钥的非压缩公钥, 共65字节， 1字节 0x04, 32字节为x坐标，32字节为y坐标
`0450863ad64a87ae8a2fe83c1af1a8403cb53f53e486d8511dad8a04887e5b2352,2cd470243453a299fa9e77237716103abc11a1df38855ed6f2ee187e9c582ba6`
其中 04是前缀
x轴坐标是 : `50863ad64a87ae8a2fe83c1af1a8403cb53f53e486d8511dad8a04887e5b2352`
y轴坐标是 : `2cd470243453a299fa9e77237716103abc11a1df38855ed6f2ee187e9c582ba6`

3. 计算公钥的 SHA-256 哈希值, 然后再hash160
得到结果 `010966776006953d5567439e5e39f86a0d273bee`
到第3步结束, 已经得到了一个pubkeyHash.

4. 取第三步结果，前面加入地址版本号（比特币主网版本号“0x00”）
得到结果 `00010966776006953d5567439e5e39f86a0d273bee`
接下来的步骤就是base58Check encoding的过程,目的是为了让我们看起来方便

5. 再对第四步结果, 进行两次SHA256
结果: `d61967f63c7dd183914a4ae452c9f6ad5d462ce3d277798075b107615c1a8a30`

6. 取第5步结果的前4个字节`d61967f6`, 加在第四步的结果最后, 作为校验
结果 : `00010966776006953d5567439e5e39f86a0d273beed61967f6`, 这个就是比特币地址的常见的16进制形态

7. 用base58, 转换一下地址展示. 
结果 `16UwLL9Risc3QfPqBUvKofHmBQ7wMtjvM`  这个就是比特币常见的地址展示

在以上几个步骤里, 我们系统里常用的地址是 第7步生成的 `16UwLL9Risc3QfPqBUvKofHmBQ7wMtjvM`, 和 第3步生成的 `010966776006953d5567439e5e39f86a0d273bee` (记为pubkey hash)
二者是可以互相转换的

### 转换代码
以下代码需要导入[bitcoinj](https://bitcoinj.github.io/)的jar包

```java
        // 第一步, 生成私钥
        String privateKey = "18E14A7B6A307F426A94F8114701E7C8E774E7F9A47E2C2035DB29A206321725".toLowerCase();
        System.out.println(privateKey.toLowerCase());

        // 第二步, 使用ECDSA计算私钥的非压缩公钥
        ECKey ecKey = ECKey.fromPrivate(HEX.decode(privateKey));
        System.out.println(ecKey.getPubKeyPoint());

        // 第三步, 先sha256, 再hash160, 得到结果 010966776006953d5567439e5e39f86a0d273bee
        System.out.println(HEX.encode(Utils.sha256hash160(HEX.decode("0450863AD64A87AE8A2FE83C1AF1A8403CB53F53E486D8511DAD8A04887E5B23522CD470243453A299FA9E77237716103ABC11A1DF38855ED6F2EE187E9C582BA6".toLowerCase()))));

        //第四步 再加上主网地址版本编号00. 得到  00010966776006953D5567439E5E39F86A0D273BEE

        // 第五步, 进行两次sha256, 得到 d61967f63c7dd183914a4ae452c9f6ad5d462ce3d277798075b107615c1a8a30
        System.out.println(HEX.encode(Sha256Hash.hashTwice(HEX.decode("00010966776006953d5567439e5e39f86a0d273bee"))));
        // 第六步, 取第5步结果的前4个字节`d61967f6`, 加在第四步的结果最后, 作为校验

        // 第七步, 对第六步的结果进行base58
        System.out.println(Base58.encode(HEX.decode("00010966776006953d5567439e5e39f86a0d273beed61967f6")));
```


## 参考文献
1. [bitcoinWiki](https://en.bitcoin.it/wiki/Technical_background_of_version_1_Bitcoin_addresses)









