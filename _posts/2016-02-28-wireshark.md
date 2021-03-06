---
layout:     post
title:      "wireshark网络分析就这么简单读书笔记"
#subtitle:   " \"分布式锁\""
date:       2016-02-28 12:00:00
author:     "gambol"
header-img: "img/post-bg-2015.jpg"
tags:
    - 读书笔记
    - 网络
    - 技术
---

# wireshark网络分析就这么简单

---

## 更新记录
- v1 2016.2.28 

## 写在前面的话
虽然我07年在计算所分析网络协议时，就用了wireshark的前身Ethereal， 但是没有想到16年的今天，还会读这个书。这个书是同事liuyue推荐我读的，很薄的一本书。两天看完。

07年做网络协议抓包分析时， 还有一个其他的选择叫CommView。 如果拿CommView和Wireshark做一个比较的话， Wireshark像是Photoshop，功能强大；CommView像是美图秀秀，使用方便简单。

这本书主要介绍了都是实用示例，不太好单独摘录出来。我挑选主要学习到的知识点来说吧。

## 知识点

### 子网、网关、子网掩码

两台电脑之间通信，有两种可能性。
- 在同一个局域网（也叫子网）内
通过ARP协议，通过IP找到MAC地址，直接通信
- 不在同一个局域网内
通过网关做转发，来进行通信

如何判断两个电脑(A和B）是不是在同一个局域网内呢？
如果A向B进行通信， 则
用A的IP地址 & A的网关   与 B的IP地址 & A的网关

### TCP重传
TCP拥塞控制和重传，是TCP协议最难的地方。如果网络性能有问题，可以看看重传和拥塞（当然，前提是查看网络是否丢包）

滑动窗口： 由接收端在ACK包里告诉发送端，还有多大的窗口可以传输。  滑动窗口的大小是可以调整的。

超时重传: 接收端发送一个包之后， 会启动一个定时器。在一段时间没有收到某个包的ACK时， 重传这个包。 重传时间设置非常影响性能。
为了减少重传时间的影响，有一个快重传协议：发送方只要一连收到三个重复确认就应当立即重传对方尚未收到的报文段，而不必继续等待设置的重传计时器时间到期。

拥塞控制： 防止网络异常时，发更多的包，加剧网络的异常。 算法核心是，改变滑动窗口的大小 一般 方案是： 慢启动+拥塞避算法。
拥塞控制算法非常多，目前linux用的是cubic。 请看下面的图 

![拥塞控制算法][1]


  [1]: http://static.zybuluo.com/gambol/g2aqml2v62myx3hdnd2zigxo/SouthEast.jpeg
