---
layout:     post
title:      "CodeCache引发的一个问题"
subtitle:   " JVM突然变慢了?"
date:       2016-03-30 12:00:00
author:     "gambol"
header-img: "img/post-bg-2016.jpg"
tags:
    - JVM
    - 技术
    - JAVA

---

# CodeCache引发的一个问题

---

## 概要
这个文章记录了一次线上问题的查询过程, 在查询过程中, 学习CodeCache的作用. CodeCache是JIT编译生成的native code的存放空间, 他的大小很可能会影响JVM的性能.

## 查询问题的经过
本文是[序列化工具对比](/2016/03/12/serialize-compare/)的后续. 问题的完整经过是

1. 线上部署了12个tomcat 实例. 但是经常出现的一个情况是`某些`tomcat实例不响应新的请求. 查看catalina日志, 提示"Dubbo Threadpool Exhausted". 其中, `某些`不固定, 可能是12个实例中的任何一个.
2. 登陆'Dubbo Threadpool Exhausted'机器上查看jstack, 发现又大量线程在Jackson 进行JSON反序列化处. 机器负载非常低. JVM有频繁的cms gc(每分钟1-2次 CMS). 通过jmap dump出来的数据发现, jackson会反序列化非常大的字符串(最大的有24M). 基于2的现象, 最初怀疑是gc造成了jvm变慢. 
3. 当时出问题的机器有一个现象, 机器访问速度慢. 如果把机器摘下线, 直接调用接口,不会出现"Dubbo Threadpool Exhausted'. 把机器放上线, 再增加一点流量, 很快就能重现线程池耗尽的现象.  必须要重启, tomcat才能恢复正常. 根据这个猜测, 猜测和gc关系不是很大.
4. 为了验证第三点的猜测, 我写了一个代码, 100并发去请求线上的接口, 很快能看见频繁的, 3-5秒钟一次的CMS GC. 但是100并发结束之后, 再用50并发去请求相同的接口, 并不会看见访问变慢的情况. 通过这个验证, 我猜想这个应该并不是因为CMS GC造成了以上的现象.
5. 关闭了线上大量使用`Jackson进行JSON反序列化处`的功能, 系统再也没有出现线程池耗尽的情况. 难道真的是Jackson 的黑锅? 
6. 根据线上日志记录, 最大的json 序列化所需要的时间大概是60S左右. 但是我自己在本地测试时, 发现即时是30M的字符串, 所需要的反序列化时间也仅仅是1S左右. --- 这个巨大的时间差是为什么呢?
7. 在线上的各个tomcat实例里, 写了一个jsp, 用同一个简单的字符串进行反序列化.  出现过线程池耗尽的`某些`机器, 速度比没有出现问题的机器要慢很多(耗尽的机器, 速度大概是200ms 没有耗尽的机器, 预计是30ms) , . 重启之后, 所有的机器速度都正常(大概是20ms). ----为什么呢?
8. google "tomcat become slow down", 发现有人出现了和我一样的问题. 请看这个[stackoverflow的链接](http://stackoverflow.com/questions/14876447/what-could-cause-global-tomcat-jvm-slowdown). 大概的意思是, 需要加上 `-XX:ReservedCodeCacheSize=256m`, 扩大CodeCache的Size.
9. 在线上选了一台机器, 加上选项`-XX:ReservedCodeCacheSize=256m`, 灰度运行了两天, 用第七步的jsp进行验证, 发现速度很快.    今天在这台机器上打开了`Jackson进行JSON反序列化处`功能, 坐等灰度的结果
10. 运行一天之后, 灰度的结果出来了, 成果不错.  现在已经在所有机器上添加了这个参数, 运行了一天后, 系统稳定.

------

## CodeCache 是什么?

要知道什么是Code Cache,  首先要知道什么是JIT(请参考我的上一篇文章, JIT介绍).  简单的来说, JIT是一种JVM优化技术, 将多次执行的热点JAVA代码编译成native code, 加快运行速度. JIT编译好之后的native code, 就存放在code cache里. CodeCache 也是一段内存空间. 

### CodeCache 和 perm区有什么区别

CodeCache 和 Permanent Generation 都属于 non-heap memory.

1. perm 区存放class的信息, 静态变量, final变量, 类的方法信息等等.
2. CodeCache 区存放的jit编译后的native code.

两段内存空间是分开的.
![JVM memory structure](http://static.zybuluo.com/gambol/kf0qnqmh6vttmofkq1onyf0e/jvm-memory-segments1.jpg)

### CodeCache满了会做什么?
随着系统运行, 越来越多的代码会被识别成**热点**代码, 存放在CodeCache里.  如果CodeCache空间不够, 则会出现"Code Cache满"的情况

#### 如何判断code cache满过?

 - tomcat可能会打印出一段日志
有些JDK7的版本, 可能并不一定会打印这个信息(我们使用的JDK7U45就没有这条日志)

> Java HotSpot(TM) 64-Bit Server VM warning: CodeCache is full. Compiler has been disabled.

 - PrintSafepointStatistics
在JVM启动参数里加上 -XX:+PrintSafepointStatistics. 能看到以下的日志

> 18867.232: HandleFullCodeCache              [     898          1              1    ]      [     1     0     2     0    14    ]  1

这个说明, JVM曾经执出现过FullCodeCache的情况, 并进行HandlFullCodeCache操作.

----

#### JVM收到CodeCache Full 信息之后, JVM 会做什么?

 当CodeCache空间被用尽之后, jvm compiler就会停止, 不再继续生成native code. 
 
 如果开启了 *UseCodeCacheFlushing* 选项,  JVM会启动code cache sweeper, 对CodeCache区进行清理工作. 具体的清理逻辑请查看[Code Cache Optimizations for Dynamically Compiled Languages](http://e-collection.library.ethz.ch/eserv/eth:8304/eth-8304-01.pdf)的第2.3.6节.
 
在直到CodeCache的剩余空间达到 *CodeCacheMinimumFreeSpace* (默认是500k) 之前, JVM在会一直关闭JIT的编译功能.

JDK6里的*UseCodeCacheFlushing* 默认为false, 在JDK7U4之后, *UseCodeCacheFlushing*默认是true. 也就是说:

 - 在JDK6里, 如果code cache满了, **默认情况下**, JVM就一直保留已经编译后的native code, 并且关闭JIT功能,不再进行新的compilation.
 - 在JDK7U4之后, 先关闭JIT, 然后对code cache清理.

但是, JDK7之后, 关于CodeCache有几个BUG

1. 即使code cache清理的很彻底了, 但是JVM仍然不会重新开启JIT功能
2. 对code cache的清理工作也能占用大量的cpu资源, 并减慢整个JVM的工作效率

这是一个已知的bug,  [JDK-8006952 : CodeCacheFlushing degenerates VM with excessive codecache freelist iteration](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=8006952). bug描述的现象和我们遇到的现象基本相同. 

还有一些其他的相关BUG列举如下:

1. [JDK-8012547 : Code cache flushing can get stuck without completing reclamation of memory](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=8012547)
2. [JDK-8029091 : Bug in calculation of code cache sweeping interval](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=8029091)

以上的这些bug在JDK8U33之后, 基本上得到了修复. 

### CodeCache的相关配置

默认情况下, CodeCache 的大小如下:

| JVM | 默认的Code cache size |
| ---- | ----|
| 32 bit client, JDK8 | 32M|
| 32-bit server with Tiered Compilation, Java 8 | 240M|
| 64-bit server with Tiered Compilation, Java 8 | 240M|
| 32-bit client, Java 7 | 32M|
| 64-bit server, Java 7 | 48M|
| 64-bit server with Tiered Compilation, Java 7 | 96M|

> 我们线上使用的是*64-bit server with Tiered Compilation*, code cache大小为96M.   

我们可以用`-XX:ReservedCodeCacheSize=XXX` 对Code Cache进行配置. 除此之外, 还有许多有用的配置

1. -XX:ReservedCodeCacheSize=size
    设置的CodeCache最大可用空间. 在JDK8里, 也可以使用 `-Xmaxjitcodesize=size`
2. -XX:CodeCacheMinimumFreeSpace=size
   最小需要预留给compiler使用的code cache 大小
3. -XX:InitialCodeCacheSize=size
   看名字就知道意思了. 初始化CodeCache大小
4. -XX:+UseCodeCacheFlushing  或者 -XX:-UseCodeCacheFlushing
   是否开启code cache 清理的功能. JDK7之后,默认都是true
5. -XX:+PrintFlagsFinal  
   在JVM启动时, 打印出所有的JVM相关参数.

对于我们线上遇到的这个问题来说, `增大ReservedCodeCacheSize空间` 或者 `关闭UseCodeCacheFlushing功能`, 都能绕过JDK7的上述的BUG.

### CodeCache的查看
在JDK7里, 对于CodeCache的查看以及管理做的不好(活该有bug), 基本上没有提供查看这部分内存使用情况的参数.JDK8里, 可以使用 PrintCodeCache 或者 PrintCodeCacheOnCompilation选项来查看CodeCache的具体使用信息 .

而JDK7里, 可以用的方法包括

1. 开启JMX之后, 使用jvisualVM 或者 jconsole查看. 
2. 在线上写代码, 调用jvm mbean方法来查看. 可以参考https://examples.javacodegeeks.com/enterprise-java/jmx/list-all-jvm-mbeans/. (我目前还没有试过, 代码待补充)

------

## 总结

从线上这个问题的解决过程中, 我们发现, 尤其对于JDK7而言,  JVM的CodeCache是一个值得注意的问题. 如果你的tomcat突然变慢, 并且恰好你的JDK版本是JDK7, 并且恰好你还开启了Tiered compliation, 你很可能命中了JDK7的这个BUG. 解决方法是, 调大ReservedCodeCacheSize 或者 关闭UseCodeCacheFlushing功能.

## 参考文献

最后,强烈推荐这些参考文献.

1. [Dynamic deoptimization 的介绍]( http://www.ibm.com/developerworks/library/j-jtp12214/)
2. http://stackoverflow.com/questions/20522870/about-the-dynamic-de-optimization-of-hotspot
3. [JVM调优](https://www.safaribooksonline.com/library/view/java-performance-the/9781449363512/ch04.html)
4. [code cache详细介绍](http://e-collection.library.ethz.ch/eserv/eth:8304/eth-8304-01.pdf)
5. [code cache full的解法](https://blogs.oracle.com/poonam/entry/why_do_i_get_message)
6. [为什么tomcat慢了呢?](http://stackoverflow.com/questions/14876447/what-could-cause-global-tomcat-jvm-slowdown)
7. [JVM7的bug描述](http://ggkekas.tumblr.com/)
8. [Codecache 调优]( https://docs.oracle.com/javase/8/embedded/develop-apps-platforms/codecache.htm)




