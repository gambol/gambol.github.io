---
layout:     post
title:      "JAVA编程指导"
#subtitle:   " \"分布式锁\""
date:       2015-08-04 12:00:00
author:     "gambol"
header-img: "img/post-bg-2015.jpg"
tags:
    - JAVA

---

# JAVA编程指导

---


这篇文章是从twitter的编程指导翻译过来的。([原文地址](https://github.com/twitter/commons/blob/master/src/java/com/twitter/common/styleguide.md))

[TOC]

## 代码风格

### 格式化

#### 什么时候应该换行？

有两个理由:

1. 超过了每行的最大长路

2. 你需要用换行来表示你的想法的停顿

写代码就像写文章，要用代码来完整表达你的意思。

#### 缩进

我们使用1BTS ([1TBS](http://en.wikipedia.org/wiki/Indent_style#Variant:_1TBS))原则，缩进2个空格。

	:::java
    // 好的示范.
    if (x < 0) {
      negative(x);
    } else {
      nonnegative(x);
    }

    // 不好的示范.
    if (x < 0)
      negative(x);

    // 也不要这样.
    if (x < 0) negative(x);
   
连续的缩进是4个空格，字符串每一行用4个空格连续。 譬如


    :::java
    // 不好的示范
    //   - 每一层随意换行.
    //   - 信息非常难以找到.
    throw new IllegalStateException("Failed to process request" + request.getId()
        + " for user " + user.getId() + " query: '" + query.getText()
        + "'");

    // 好的示范.
    //   - 代码自解释.
    //   - 添加代码也比较容易.
    throw new IllegalStateException("Failed to process"
        + " request " + request.getId()
        + " for user " + user.getId()
        + " query: '" + query.getText() + "'");

如果方法名字较长时，如何声明呢？

   	 :::java
    // 随意摆放各个变量.不在乎每一行的字母数量
    String downloadAnInternet(Internet internet, Tubes tubes,
        Blogosphere blogs, Amount<Long, Data> bandwidth) {
      tubes.download(internet);
      ...
    }

    // 可以接受的.
    String downloadAnInternet(Internet internet, Tubes tubes, Blogosphere blogs,
        Amount<Long, Data> bandwidth) {
      tubes.download(internet);
      ...
    }

    // 好一点, 在声明和代码之间有空格分开.
    String downloadAnInternet(Internet internet, Tubes tubes, Blogosphere blogs,
        Amount<Long, Data> bandwidth) {

      tubes.download(internet);
      ...
    }

    // 也不错,不过如果变量名过长，很容易看不到括号
    public String downloadAnInternet(Internet internet,
                                     Tubes tubes,
                                     Blogosphere blogs,
                                     Amount<Long, Data> bandwidth) {
      tubes.download(internet);
      ...
    }

    // 最推荐的方式，声明和变量分开，有空格区分.
    public String downloadAnInternet(
        Internet internet,
        Tubes tubes,
        Blogosphere blogs,
        Amount<Long, Data> bandwidth) {

      tubes.download(internet);
      ...
    }

##### 链式函数调用

    :::java
    // 不好.
    //   - 代码不讲究.
    Iterable<Module> modules = ImmutableList.<Module>builder().add(new LifecycleModule())
        .add(new AppLauncherModule()).addAll(application.getModules()).build();

    // 好一点.
    //   - 每一层逻辑是分开的.
    //   - 不过，点号和方法调用没有放在一起
    Iterable<Module> modules = ImmutableList.<Module>builder().
        add(new LifecycleModule()).
        add(new AppLauncherModule()).
        addAll(application.getModules()).
        build();

    // Good.
    //   - 每一行都是一个完整的逻辑.
    //   - 清楚知道每一行代码的调用.
    Iterable<Module> modules = ImmutableList.<Module>builder()
        .add(new LifecycleModule())
        .add(new AppLauncherModule())
        .addAll(application.getModules())
        .build();
 
#### 不用tab

tab 在每个IDE里表现不一样。不要用tab做分割哦


#### 结尾不要无谓的空格

### 变量，类的名字和方法名字

##### 修饰符的顺序

参考 [Java Language Specification](http://docs.oracle.com/javase/specs/) [8.1.1](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.1.1),[8.3.1](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.3.1) 和[8.4.3](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.4.3)).

    :::java
    // 不好的顺序.
    final volatile private String value;

    // 好的顺序.
    private final volatile String value;
    
### 变量命名

#### 不要用超短的变量名

#### 如果需要，变量名里要包含单位

    :::java
    // 差的.
    long pollInterval;
    int fileSize;

    // 好的.
    long pollIntervalMs;
    int fileSizeGb.

    // 更好的.
    //   - Unit is built in to the type.
    //   - The field is easily adaptable between units, readability is high.
    Amount<Long, Time> pollInterval;
    Amount<Integer, Data> fileSize;

#### 不要再变量里增加源信息(metadata)

变量名应该是说明这个变量干嘛的，不要再变量名里添加额外的信息，譬如变量范围或者变量类型。

    :::java
    // 不好的. 
    // - 因为他包含了使用场景和类型
    Map<Integer, User> idToUserMap;
    String valueString;

    // 好.
    Map<Integer, User> usersById;
    String value;
    
也不要在变量里包含变量的作用域。建议如果比较复杂，就把他分成多个函数的部分。 （注：怎么这么严格）

    :::java
    // Bad.
    String _value;
    String mValue;

    // Good.
    String value;

### 操作符之间要留有空格

### 如果复杂的操作符之间，记得用括号表示优先级
    :::java
    // 不好.
    return a << 8 * n + 1 | 0xFF;

    // 好.
    return (a << (8 * n) + 1) | 0xFF;
    
操作顺序写的越明确越好
    :::java
    if ((values != null) && (10 > values.size())) {
      ...
    }

### 文档

如果代码想要被用的次数多，好文档必不可少

#### 明确写出完整的注释。不要只说半句话

    :::java
    // Bad.
    /**
     * 这是一个关于cache的类。他能做缓存
     */
    class Cache {
      ...
    }

    // Good.
    /**
     * A volatile storage for objects based on a key, which may be invalidated and discarded.  （注：不好翻译）
     */
    class Cache {
      ...
    }
    
#### 如何为类写注释

类的注释的目的是消灭掉你API里可能不好表达的概念性问题，帮助别人快速而又*准确*的使用你的API。 好的类文档通常包含一句话概括，如果有必要的话，也可以详细展开。

    :::java
    /**
     * RPC里类似于tee的类，能保证每一个RPC的输入，在RPC call返回之前，输出到tee的两端
     *
     * @param <T> The type of the tee'd service.
     */
    public class RpcTee<T> {
      ...
    }

#### 如何为方法写注释
方法注释应该告诉别人，这个方法是干什么的. 根据方法输入的类型，有时候需要详细描述函数的输入

  :::java
    // 不好.
    //   - 说了和没有说一样
    /**
     * Splits a string.
     *
     * @param s A string.
     * @return A list of strings.
     */
    List<String> split(String s);

    // 好一些.
    //   - 能知道这个函数是干嘛的
    //   - 但是仍然有一些不确定的点.
    /**
     * 根据空格分隔字符串.
     *
     * @param s 被分隔的字符串.   {@code null} null串和空串相同的处理逻辑.
     * @return 用空格分隔的字符串形成的list.
     */
    List<String> split(String s);

    // 棒.
    //   -  包含了每一个边界条件.
    /**
     * 根据空格分隔字符串.  重复空格会被当做一个空格.
     *
     * @param s 被分隔的字符串.   {@code null} null串和空串相同的处理逻辑.
     * @return 用空格分隔的字符串形成的list.
     */
    List<String> split(String s);
    
#### 专业点
不要在注释里写幼稚的话~

#### 不要重复写override里已经写过的话
如果接口里已经描述了这个内容，那么，如果你Overrider这个方法，应该新增一些额外的说明

#### 使用javadoc的特性

##### 不要用author这个标签
commit记录比author标签更靠谱

### Import

#### Import的顺序

先应用高级别的package，用空行分隔。 static的import放在最后

    :::java
    import java.*
    import javax.*

    import scala.*

    import com.*

    import net.*

    import org.*

    import com.twitter.*

    import static *
    
#### 不要用用星号引用

星号会造成大家看不到引用的来源
    :::java
    // Bad.
    //   - Foo 是从哪里来的呢?
    import com.twitter.baz.foo.*;
    import com.twitter.*;

    interface Bar extends Foo {
      ...
    }

    // Good.
    import com.twitter.baz.foo.BazFoo;
    import com.twitter.Foo;

    interface Bar extends Foo {
      ...
    }

###  聪明的使用注解

#### @Nullable
通常来说，默认每个变量都不能是null的。如果允许返回或者输入null，请用这个注解来显示标明

    :::java
    class Database {
      @Nullable private Connection connection;

      @Nullable
      Connection getConnection() {
        return connection;
      }

      void setConnection(@Nullable Connection connection) {
        this.connection = connection;
      }
    }

#### @VisibleForTesting

### 使用接口(interface)
接口将实现和声明分离，是一个很好的编程习惯。提供接口出去，然后隐藏你的实现

如果接口数目过多，可以考虑下面的风格

    :::java
    interface FileFetcher {
      File getFile(String name);

      // 既使用了interface的优点，又不会造成类膨胀
      // 如果某个interface只有一个实现时，可以考虑这种方法
      static class HdfsFileFetcher implements FileFetcher {
        @Override File getFile(String name) {
          ...
        }
      }
    }

#### 考虑使用现有的interface
有时候，使用现有的接口可以让你的类更加容易被人使用。

    :::java
    // 没有过多的考虑。如果别人想用Blobs，还需要写胶水代码(glue code)
    class Blobs {
      byte[] nextBlob() {
        ...
      }
    }

    // Much better.  
    // 调用者可以用标准的collection函数来声明，也可以做类似于filtering这种复杂的操作
    class Blobs implements Iterable<byte[]> {
      @Override
      Iterator<byte[]> iterator() {
        ...
      }
    }
    
注意：  不要在已经正常工作的接口上扩展接口哦。

## 最佳实践

### 防御式编程

#### 不要用assert
assert可以在执行时被关闭。推荐使用guava的preconditions

#### Preconditions
按照常规来说，公共(public)方法或者构造函数，都需要检查输入是否为null（除非显示标明这个输入可以是nullable)

    :::java
    // Bad.
    //   - 如果callback为null，或者file为null，我们会很晚才发现这个异常
    class AsyncFileReader {
      void readLater(File file, Closure<String> callback) {
        scheduledExecutor.schedule(new Runnable() {
          @Override public void run() {
            callback.execute(readSync(file));
          }
        }, 1L, TimeUnit.HOURS);
      }
    }

    // Good.
    class AsyncFileReader {
      void readLater(File file, Closure<String> callback) {
        checkNotNull(file);
        checkArgument(file.exists() && file.canRead(), "File must exist and be readable.");
        checkNotNull(callback);

        scheduledExecutor.schedule(new Runnable() {
          @Override public void run() {
            callback.execute(readSync(file));
          }
        }, 1L, TimeUnit.HOURS);
      }
    }
    
#### 最小化代码的可见性
只把想让别人使用的接口或者变量暴露出去。如果你想写一个线程安全的代码，这一点尤其重要。

#### 善用imutable
Mutable的变量经常会给你带来负担，你需要小心你传给别人的mutable对象是不是被人修改了。

    :::java
    // Bad.
    //   - 所有人可以修改生日
    //   - 调用getAttributes，就可以获得底层的map，就可以任意修改.
    public class User {
      public Date birthday;
      private final Map<String, String> attributes = Maps.newHashMap();

      ...

      public Map<String, String> getAttributes() {
        return attributes;
      }
    }

    // Good.
    public class User {
      private final Date birthday;
      private final Map<String, String> attributes = Maps.newHashMap();

      ...

      public Map<String, String> getAttributes() {
        return ImmutableMap.copyOf(attributes);
      }

      // 如果你觉得别人不需要知道所有的map里的信息，那么你就可以考虑单独返回一些对象.
      @Nullable
      public String getAttribute(String attributeName) {
        return attributes.get(attributeName);
      }
    }

#### 小心null
最好是使用`Optional`, 不太推荐使用null
    
#### try catch之后，记得finally
    
### Clean code
#### 不要让人误导
如果是Long型，那么就写成100L,而不是100l。

#### 移除dead 代码

#### 使用更加general的类型
声明变量或者函数时， 更可能声明更general的类型。可以避免在你接口里暴露细节

    :::java
    // Bad.
    //   - Implementations of Database must match the ArrayList return type.
    //   - Changing return type to Set<User> or List<User> could break implementations and users.
    interface Database {
      ArrayList<User> fetchUsers(String query);
    }

    // Good.
    //   - Iterable defines the minimal functionality required of the return.
    interface Database {
      Iterable<User> fetchUsers(String query);
    }

#### 避免类型转换
类型转换(typecasting)是poor代码的表现。

    ::: java
    // Bad
    Person person = (Person)boy;

#### 多使用final的变量


#### 异常处理
##### 只捕获应该捕获的异常
不要一次捕获所有的异常。这是偷懒

    :::java
    // Bad.
    //   - If a RuntimeException happens, the program continues rather than aborting.
    try {
      storage.insertUser(user);
    } catch (Exception e) {
      LOG.error("Failed to insert user.");
    }

    try {
      storage.insertUser(user);
    } catch (StorageException e) {
      LOG.error("Failed to insert user.");
    }

##### 不要吞掉异常
绝不要catch里，什么都不处理。

##### When interrupted, reset thread interrupted state
有一些block的操作时，会抛出InterruptedException。 如果你接收到InterruptedException，最佳实践是保留线程的interrupted state

IBM有一篇关于这个很好的[文章](http://www.ibm.com/developerworks/java/library/j-jtp05236/index.html)

    :::java
    // Bad.
    //   - 周围或者高层级的代码不知道这个线程被中断了
    try {
      lock.tryLock(1L, TimeUnit.SECONDS)
    } catch (InterruptedException e) {
      LOG.info("Interrupted while doing x");
    }

    // Good.
    //   -  保留线程的中断状态
    try {
      lock.tryLock(1L, TimeUnit.SECONDS)
    } catch (InterruptedException e) {
      LOG.info("Interrupted while doing x");
      Thread.currentThread().interrupt();
    }

##### 抛出合适的异常
让你的API用户捕获异常，而不要只是抛出最高级别的Exception。
即使你调用的代码里，抛出的异常是Exception，你也需要注意不要让这个Exception到过于上层。你要努力让你的调用者不需要太关心你的异常

    :::java
    // Bad.
    //   - 抛出一个Exception，让别人很难受
    interface DataStore {
      String fetchValue(String key) throws Exception;
    }

    // Better.
    //   - 让上层知道了你的具体实现细节.
    interface DataStore {
      String fetchValue(String key) throws SQLException, UnknownHostException;
    }

    // Good.
    //   - 封装了你的实现.
    //   - 不需要让别人针对你的异常做不同的处理
    interface DataStore {
      String fetchValue(String key) throws StorageException;

      static class StorageException extends Exception {
        ...
      }
    }

### 使用新的api和新的库

#### 用StringBuilder 代替 StringBuffer
[StringBuffer](http://docs.oracle.com/javase/7/docs/api/java/lang/StringBuffer.html) is 线程安全的,但是很多时候不需要线程安全

#### 使用ScheduledExecutorService ，而不用 Timer
参考<Java Concurrency in Practice>  和stackoverflow上的这个
[问题](http://stackoverflow.com/questions/409932/java-timer-vs-executorservice)).

- `Timer` 只有一个线程，如果有个任务执行时间太长，就会影响下面的线程执行

- `ScheduledThreadPoolExecutor` 可以配置成多个线程和一个`ThreadFactory`. 

-  `TimerTask` 的异常会杀掉这个线程, 这样`Timer` 就挂了。

- ThreadPoolExecutor 还提供 `afterExceute`方法，可以让你方便的处理定时任务的结果.

#### 用List ，不用 Vector
`Vector` 是同步的。你一般也用不到. 如果要用同步的集合,jdk7里也有很多 [替代品](http://docs.oracle.com/javase/7/docs/api/java/util/Collections.html#synchronizedList(java.util.List))

### equals() and hashCode()
如果你覆盖了某个类里equals() and hashCode()中一个方法，那么不要忘了覆盖另外一个
*equals/hashCode 相关文献参考
[contract](http://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#hashCode())*

[Objects.equal()](http://docs.guava-libraries.googlecode.com/git-history/v11.0.2/javadoc/com/google/common/base/Objects.html#equal(java.lang.Object, java.lang.Object))
and
[Objects.hashCode()](http://docs.guava-libraries.googlecode.com/git-history/v11.0.2/javadoc/com/google/common/base/Objects.html#hashCode(java.lang.Object...))

### 不要过早优化
参考Donald Knuth 关于[过早优化](http://c2.com/cgi/wiki?PrematureOptimization) 的意见

通常情况下，最好的办法是先上一个性能不一定那么好的版本，除非你有强烈的预感，必须要做这个优化。

### 关于TODOs

#### 尽早写todo
TODO不是一个坏习惯。留下TODO可以告诉别人这里有什么问题，你当时是怎么考虑的。

#### 每个TODO都要留下相关人的名字
如果没有名字，估计这个TODO永远不会再DO了

    :::java
    // Bad.
    //   - TODO 没有留下名字.
    // TODO: Implement request backoff.

    // Good.
    // TODO(George Washington): Implement request backoff.

#### 领养 TODOs
如果TODO的owner已经离开这个公司/项目啦， 或者如果你已经根据TODO做了一些修改，那么这个TODO就归你啦。

### 遵守迪米特法则([迪米特法则](http://en.wikipedia.org/wiki/Law_of_Demeter))
迪米特法则又叫最少知道原则，就是说一个对象应当对其他对象有尽可能少的了解。

#### 在类里面
只关心你要的东西，不要牵扯太多。核心点是，推迟代码层面的实现，只要能满足你要求的最少的接口。

    :::java
    // Bad.
    //   - Weigher要了hosts 和 port，但是这两个参数只用来传递给别的service.
    class Weigher {
      private final double defaultInitialRate;

      Weigher(Iterable<String> hosts, int port, double defaultInitialRate) {
        this.defaultInitialRate = validateRate(defaultInitialRate);
        this.weightingService = createWeightingServiceClient(hosts, port);
      }
    }

    // Good.
    class Weigher {
      private final double defaultInitialRate;

      Weigher(WeightingService weightingService, double defaultInitialRate) {
        this.defaultInitialRate = validateRate(defaultInitialRate);
        this.weightingService = checkNotNull(weightingService);
      }
    }

（注：不知道这句话怎么翻译，大概意思是说，好的构造函数可以使用工厂方法或者builder，但是传递一个底层的service进去，更加有利于你做单元测试，如果代码发生变更，也可以做到对你透明）
If you want to provide a convenience constructor, a factory method or an external factory
in the form of a builder you still can, but by making the fundamental constructor of a
Weigher only take the things it actually uses it becomes easier to unit-test and adapt as
the system involves.

#### 方法
如果一个方法有多个块，可以考虑把他们抽取出来，让每个块只做一件事情。这么做的好处是，可以让人看到你的代码时，更像是在读一篇文章。抽取出来的块也更加适合人类阅读。
下面这个是一个很典型的坏代码

    :::java
    void calculate(Subject subject) {
      double weight;
      if (useWeightingService(subject)) {
        try {
          weight = weightingService.weight(subject.id);
        } catch (RemoteException e) {
          throw new LayerSpecificException("Failed to look up weight for " + subject, e)
        }
      } else {
        weight = defaultInitialRate * (1 + onlineLearnedBoost);
      }

      // Use weight here for further calculations
    }

推荐写法是:

    :::java
    void calculate(Subject subject) {
      double weight = calculateWeight(subject);

      // Use weight here for further calculations
    }

    private double calculateWeight(Subject subject) throws LayerSpecificException {
      if (useWeightingService(subject)) {
        return fetchSubjectWeight(subject.id)
      } else {
        return currentDefaultRate();
      }
    }

    private double fetchSubjectWeight(long subjectId) {
      try {
        return weightingService.weight(subjectId);
      } catch (RemoteException e) {
        throw new LayerSpecificException("Failed to look up weight for " + subject, e)
      }
    }

    private double currentDefaultRate() {
      defaultInitialRate * (1 + onlineLearnedBoost);
    }

看你代码的人，可以很清楚的看到你是怎么计算重量的。如果他想要看具体实现，代码往下拖就行啦

### 不要重复自己 ([DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself))
写代码的一个重要原则是，不要重复自己。可以参考这个
[文章](http://c2.com/cgi/wiki?DontRepeatYourself).

#### 如果这个常量是有意义的，一定要提取出来

#### 重复的代码，要提取出来

### 管理好你的线程
开启一个新线程时，需要特别关心这个线程的生命周期。自己要熟悉守护线程(daemon thread)和 非守护线程的区别（可以参考这篇文章 [Thread](http://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html)。
如果不了解这些概念，可能会让你的代码在shutdown时被hang住。

当正在运行的线程都是守护线程时，Java 虚拟机退出。
必须在启动线程前调用，调用setDaemon方法，将线程置为守护线程

Shutting down [ExecutorService](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html) 是一个有点tricky的过程。如果你用非守护线程管理executor service，那么你至少需要遵守这种方法。（注：ExecutorService关闭时，先调用shutdown()，不再接受所有新的任务，然后调用shutdownNow（）来去掉还遗留的任务)


如果你想vm在shutdown 时，自动给你进行清理工作，可以考虑调用[ShutdownRegistry](https://github.com/twitter/commons/blob/master/src/java/com/twitter/common/application/ShutdownRegistry.java)进行注册.


### 删掉没用的代码
#### 删掉完全没用的临时变量

    :::java
    // Bad.
    List<String> strings = fetchStrings();
    return strings;

    // Good.
    return fetchStrings();

#### 删掉没意义的赋值

    :::java
    // Bad.
    //   - 这个null从来没有用过.
    String value = null;
    try {
      value = "The value is " + parse(foo);
    } catch (BadException e) {
      throw new IllegalStateException(e);
    }

    // Good
    String value;
    try {
      value = "The value is " + parse(foo);
    } catch (BadException e) {
      throw new IllegalStateException(e);
    }

### 不要用fast 或者 optimized作为命名
你的API里的fast 或者 optimized 只能迷惑人，不能起到任何作用
Don't bewilder your API users with a 'fast' or 'optimized' implementation of a method.

    :::java
    int fastAdd(Iterable<Integer> ints);

    // 有fastAdd啦，还用add干嘛。。
    int add(Iterable<Integer> ints);

