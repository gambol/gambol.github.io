---
layout:     post
title:      "Spring \"no matching editors or conversion strategy found\"的问题解决 以及 Spring AOP代理的介绍"
subtitle:   "解决问题引发出来的思考"
date:       2016-06-20 00:00:00
author:     "gambol"
header-img: "img/post-bg-2016.jpg"
tags:
    - JAVA
    - 技术
---


## 问题
昨天晚上,在线上调试时, 发现在一个spring的bean里, 只要加上`@Transactional`注解后, 就会在spring启动时报错

报错信息有点多, 需要注意的点只有一点

```
Cannot convert value of type [com.sun.proxy.$Proxy21 implementing gambol.examples.spring.service.Console,org.springframework.aop.SpringProxy,org.springframework.aop.framework.Advised] to required type [gambol.examples.spring.service.AbstractConsole] for property 'abstractConsole': no matching editors or conversion strategy found
```

## 解决方法

先说解决方案, 增加一行配置, 解决

```
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

强制使用cglib进行代理.

## 解决过程 

- 早上看完报错信息之后, liuyue猜测是因为使用了transactional之后, 生成的是jdk的proxy, jdk的proxy不能注入成AbstractConsole.
- 我有疑问, 为什么使用了transactional注解之后才生成代理? 难道不是所有spring管理的service都会生成一个代理么?
- 我把线上的代码层次拷贝到了本地,生成的代码结构是

![image_1alrm8q5j16mtp0510m143io7l9.png-32.3kB](http://static.zybuluo.com/gambol/dmis2hbypu7g0o9yqm70dbkr/image_1alrm8q5j16mtp0510m143io7l9.png)

其中, consoleService会通过spring 注入到contanerService里.

```
  <bean id="containerService" class="gambol.examples.spring.service.ContainerService">
    <property name="abstractConsole" ref="consoleService"/>
  </bean>

```

ConsoleService的代码是

```
@Service
public class ConsoleService extends AbstractConsole {

    @Transactional
    public void test() {
        System.out.println(" test in consoleService");
       // console();
    }

}
```

- 在本地启动, 问题无法重现. debug时发现, 当去掉`@Transactional`注解后, ConsoleService并没有生成任何代理. 加上`@Transactional`注解后, ConsoleService使用CGLib生成代理.
- 突然发现, 我的Console这个接口里并没有任何方法, 在Console加上方法, 再重启tomcat, 问题复现.

```
public interface Console {
    // 这个方法很关键
    void console();
}
```

- 到现在为止, 线上问题的原因已经确认. **jdk的proxy不能注入成AbstractConsole.**
- 解决方案是:   强制使用CGlib来生成代理.(Spring官方的说法是, JDK生成的代理, 比cglib的代理稍微快一点, 所以会默认使用jdk生成的基于接口的代理)
- 引入了一个新的问题: 为什么Console里没有方法时, spring就会使用cglib进行代理, spring是怎么判断的?
- debug spring的启动过程, 最后在`ProxyProcessorSupport.java`里找到原因.

```
  protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
    Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
    boolean hasReasonableProxyInterface = false;
    for (Class<?> ifc : targetInterfaces) {
      if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
      // Spring会在这段代码里, 判断需要代理的接口里的方法数量
          ifc.getMethods().length > 0) {
        hasReasonableProxyInterface = true;
        break;
      }
    }
    if (hasReasonableProxyInterface) {
      // Must allow for introductions; can't just set interfaces to the target's interfaces only.
      for (Class<?> ifc : targetInterfaces) {
        proxyFactory.addInterface(ifc);
      }
    }
    else {
      proxyFactory.setProxyTargetClass(true);
    }
  }
```


##  结论/收获

1. spring并不会为每个bean,都生成代理类
2. 只有使用AOP时, spring才会为他生成代理类(proxy class)
3. 生成代理类的规则

|    方式     | proxy-target-class="false"(默认情况)   |  proxy-target-class="true"  |
| --------   | -----  | ----  |
| bean实现了一个非空接口 | 使用JDK proxy 生成接口的代理 |   CGLIB  |
| bean未实现一个非空接口 | CGLIB代理生成实体bean | CGLIB|

## 参考文献
1. [Spring AOP: CGLIB or JDK Dynamic Proxies?](http://cliffmeyers.com/blog/2006/12/29/spring-aop-cglib-or-jdk-dynamic-proxies.html)
2. [Spring官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop-api.html#aop-pfb-proxy-types)