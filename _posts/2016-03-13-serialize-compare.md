---
layout:     post
title:      "序列化工具对比"
subtitle:   "比比比,不怕不识货就怕货比货"
date:       2016-03-12 12:00:00
author:     "gambol"
header-img: "img/post-bg-2016.jpg"
tags:
    - 序列化
    - 技术
---


# 序列化工具对比

## 起因
今天早上凌晨, 机器报警. jstack之后, 发现大部分线程都在进行json反序列化. 一看代码, 发现是因为: 我们的逻辑是从缓存中取出JSON字符串, 反序列化成object.

 绝大部分线程都在下面的第二个语句里.


```JAVA
String cacheDetail = cacheInfo.queryCacheDetail();
AdapterHotelProduct adapterHotelProduct = JsonUtil.parseObject(cacheDetail, new TypeReference<AdapterHotelProduct>() {
        });
```

我猜想是利用Jackson 进行JSON反序列化速度太慢, 因此,我想看看有哪些序列化工具, 以及对各种工具性能进行对比. 

没有耐心的同学, 可以直接看结论.:D

## 经过
经过就是, 我用一天的时间写了一个benchmark代码,进行测试. 测试代码请看 [github](https://github.com/gambol/javaStudy/tree/master/examples/core/main/java/gambol/examples/serialize/benchmark)

大致原理是:
针对同一个对象, 用各种工具进行15 * 1000次序列化和反序列化, 分别取最大,最小,平均值进行对比. 此外, 还要对比每个工具序列化之后的大小.
另外, 为了防止对象大小引起的误差, 我还分别对比了各种大小的对象.

其实, 在 [这个链接](https://github.com/eishay/jvm-serializers/wiki)里, 已经有比较成熟的结论了. 我在参考了这个结论后, 自己挑选了一些进行验证. 

接下来, 我简单介绍一下我对比的这些序列化工具.

### Jackson
jackson 是目前(2016年)最成熟的json转化工具. 我用过gson, fastjson, 但是gson和fastjson都遇到了一些bug和性能问题, 最后全面投向了jackson.

将object转换成json, 有一个非常大好处是: 肉眼可读, 更是垮语言的

### java自带序列化工具
额,没啥好说的, java自己带的序列化工具

### kryo 
因为java序列化工具相对较慢, 所以产生了kryo和FST 这些高效的序列化工具. 具体可以参考[blog](http://blog.hazelcast.com/kryo-serializer/)

kryo 只能在java里使用

### hessian
[hessian](http://hessian.caucho.com/) 是一个跨语言的序列化包. 支持好几种语言. 奇怪的是, hessian代码在2013年之后就没有更新了.  dubbo里使用的hessian 是alibaba 自己改过的.

等我做完测试之后, 我很奇怪,为什么要用hessian?  (莫非是我测试代码有问题, 或者我没有参透hessian)

### protobuf
[protobuf](https://github.com/google/protobuf) 目前最著名的跨语言序列化工具,google 出品, 品质绝对有保证. 目前最新的版本是 3.0 beta 2, 但是我为了稳定, 在测试时使用的版本是2.6.1.

protobuf最大的优点是, 速度快, 压缩比例高. 缺点是, 要手写proto文件, 并且必须要protobuf生成bean文件.

### protostuff
[protostuff](https://github.com/protostuff/protostuff)是一个java序列化lib, 可以支持序列化成json, protobuf, xml, kvp,yaml等等.  
看名字, 我觉得protostuff是为了解决protobuf的缺点, 才引入的. protostuff解决了proto文件的问题, 它不需要依赖.proto文件，可以直接对普通的javabean进行序列化、反序列化的操作, 并且速度还比较快.


## 测试数据

### 测试环境
- mac osx 10.11.3
- 8G 1600Mhz DDR
- i5 2.7G
- JDK 8(1.8.0_25)
- Jackson版本 2.4.0
- hessian版本 4.0.33
- protobuf版本 2.6.1
- protostuff版本 1.3.5
- kryo版本 3.0.3

### 测试结果

#### 序列化

- 小体积对象

| 工具 | 最大时间(milliseconds) | 最小时间 | 平均时间 | 大小 (字节) |
|------|------|------|------|-----|
| jackson | 32 | 12 | 15 | 3004 |
| java自带 | 51 | 20 | 28 | 1521|
|kryo |35 | 7| 12 | 1012|
| hessian | 117 |97|103|1056|
| protobuf | 27 | 2 | 8 | 1043 |
| protostuff | 10 | 4 | 6 | 1043 |

---

- 大体积对象

| 工具 | 最大时间(milliseconds) | 最小时间 | 平均时间 | 大小 (字节) |
|------|------|------|------|-----|
| jackson | 422 | 357 | 376 | 94226 |
| java自带 | 381 | 365 | 371 | 38183|
|kryo |199 | 192| 192 | 30731|
| hessian | 364 |322|335|29783|
| protobuf | 105 | 88 | 93 | 33737 |
| protostuff | 159 | 155 | 156 | 33737 |

----

#### 反序列化


- 小体积对象

| 工具 | 最大时间(milliseconds) | 最小时间 | 平均时间 |
|------|------|------|------|
| jackson | 48 | 20 | 24 |
| java自带 | 59 | 28 | 34 | 
|kryo |36 | 10| 17 |
| hessian | 29 |16|19|
| protobuf | 10 | 4 | 6 | 
| protostuff | 14 | 5 | 6 | 

----
- 大体积对象

| 工具 | 最大时间(milliseconds) | 最小时间 | 平均时间 |
|------|------|------|------|
| jackson | 661 | 639 | 648 |
| java自带 | 411 | 387 | 394 | 
|kryo |217 | 207| 209 | 
| hessian | 354 |344|350|
| protobuf | 148 | 141 | 143 | 
| protostuff | 175 | 167 | 169 | 


## 结论
1. hessian 相对来说最差(谨慎怀疑, 是我的姿势不对)
2. 用jackson的速度是最慢的. 但是很多时候, 这个性能可以接受.
3. protobuf 牛逼.
4. protostuff 性能逼近protobuf, 不知道有没有公司商业中使用了protostuff. 这个软件值得一试
5. kryo 其实也还不错. 如果protostuff有bug, 可以试用kryo

## 附录
完整的测试结果

     gambol.examples.serialize.benchmark.Main
                              test      min      max      avg  (MILLISECONDS)
    -----------------------------------------------------------------
    
    with 0 children
              jackson[0] serialize        1       27        4
            jackson[0] deserialize        2       29       10
         protobuf[0] serialization        0        7        1
       protobuf[0] deserialization        0        6        1
                 java[0] serialize        3       25        5
               java[0] deserialize       15       50       20
                 kryo[0] serialize        1       11        2
               kryo[0] deserialize        1        7        2
              hessian[0] serialize       84      165      105
            hessian[0] deserialize        9       38       15
           protostuff[0] serialize        0        6        1
         protostuff[0] deserialize        0        3        0
    
    encoded sizes:
                        jackson[0]   88
                       protobuf[0]   28
                           java[0]   325
                           kryo[0]   26
                        hessian[0]   168
                     protostuff[0]   28
    
    
    with 32 children
             jackson[32] serialize       12       32       15
           jackson[32] deserialize       20       48       24
        protobuf[32] serialization        2       27        8
      protobuf[32] deserialization        4       10        6
                java[32] serialize       20       51       28
              java[32] deserialize       28       59       34
                kryo[32] serialize        7       35       12
              kryo[32] deserialize       10       36       17
             hessian[32] serialize       97      117      103
           hessian[32] deserialize       16       29       19
          protostuff[32] serialize        4       10        6
        protostuff[32] deserialize        5       14        6
    
    encoded sizes:
                       jackson[32]   3004
                      protobuf[32]   1043
                          java[32]   1521
                          kryo[32]   1012
                       hessian[32]   1056
                    protostuff[32]   1043
    
    
    with 64 children
             jackson[64] serialize       22       25       23
           jackson[64] deserialize       39       41       40
        protobuf[64] serialization        5        6        5
      protobuf[64] deserialization        8       11        9
                java[64] serialize       24       27       25
              java[64] deserialize       38       41       39
                kryo[64] serialize       12       13       12
              kryo[64] deserialize       13       20       14
             hessian[64] serialize      105      113      109
           hessian[64] deserialize       26       29       27
          protostuff[64] serialize        9       10       10
        protostuff[64] deserialize       10       13       11
    
    encoded sizes:
                       jackson[64]   5916
                      protobuf[64]   2067
                          java[64]   2673
                          kryo[64]   1940
                       hessian[64]   1953
                    protostuff[64]   2067
    
    
    with 1024 children
           jackson[1024] serialize      357      422      376
         jackson[1024] deserialize      639      661      648
      protobuf[1024] serialization       88      105       93
    protobuf[1024] deserialization      141      148      143
              java[1024] serialize      365      381      371
            java[1024] deserialize      387      411      394
              kryo[1024] serialize      192      199      196
            kryo[1024] deserialize      207      217      209
           hessian[1024] serialize      322      364      335
         hessian[1024] deserialize      344      354      350
        protostuff[1024] serialize      155      159      156
      protostuff[1024] deserialize      167      175      169
    
    encoded sizes:
                     jackson[1024]   94226
                    protobuf[1024]   33737
                        java[1024]   38183
                        kryo[1024]   30731
                     hessian[1024]   29783
                  protostuff[1024]   33737






