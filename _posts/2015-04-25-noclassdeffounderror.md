---
layout: post
title: 记一次上线的NoClassDefFoundError异常
category: program
---

上周同事上线一个dubbo服务，当然上线之前经过功能测试和压测，都是没有问题的，上线之后服务能够启动，但是无法提供服务，报`NoClassDefFoundError`，后面临时加上日志，才找到问题。

#### 现象
这个新上线的服务分别向两个业务提供服务，一个是http服务，另外一个是rpc服务，处于性能考虑，将前者的http服务也改为rpc，又为了方便管理，统一了facade接口，所以出现问题时的第一直觉是这个接口jar包有问题，导致接口不兼容，后来证实时错误的。部分异常日志如下：

~~~~

java.lang.NoClassDefFoundError: Could not initialize class com.x.x.XManager
        at com.x.dubbo.impl.XDubboImpl.transferSso(XDubboImpl.java:40) ~[YYY.20150414.jar:na]
        at com.alibaba.dubbo.common.bytecode.Wrapper1.invokeMethod(Wrapper1.java) ~[na:2.5.3]
        at com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory$1.doInvoke(JavassistProxyFactory.java:46) ~[dubbo-2.5.3.jar:2.5.3]
        at com.alibaba.dubbo.rpc.proxy.AbstractProxyInvoker.invoke(AbstractProxyInvoker.java:72) ~[dubbo-2.5.3.jar:2.5.3]
        at com.alibaba.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:53) ~[dubbo-2.5.3.jar:2.5.3]
        at com.alibaba.dubbo.rpc.protocol.dubbo.filter.TraceFilter.invoke(TraceFilter.java:78) ~[dubbo-2.5.3.jar:2.5.3]
        at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91) [dubbo-2.5.3.jar:2.5.3]

~~~~

平时遇到比较多的是`ClassNotFoundException`，`NoClassDefFoundError`接触比较少，这次为什么遇到，比较奇怪。

#### 分析

出问题之后，特意查了一下以上两种异常的区别，javadoc解释如下：

*NoClassDefFoundError*

>
>Thrown if the Java Virtual Machine or a ClassLoader instance tries to load in the definition of a class (as part of a normal method call or as part of creating a new instance using the new expression) and no definition of the class could be found.
The searched-for class definition existed when the currently executing class was compiled, but the definition can no longer be found.

*ClassNotFoundException*

>Thrown when an application tries to load in a class through its string name using:
>
* The forName method in class Class.
* The findSystemClass method in class ClassLoader .
* The loadClass method in class ClassLoader.
>
>but no definition for the class with the specified name could be found.

两者的区别在于，后者是缺少.class文件，比如缺少依赖的jar包，而前者并不缺少依赖包，只是找不到类定义，如果出现程序包了后面的异常，一般需要检查类的初始化部分，比如类属性定义和static代码块，如果找不到问题，建议在类初始化代码块中添加try-catch，并将异常打印出来，这样定位问题就简单一些。

ok，分析之后就是证实，经过查看代码，确认线上的`XManager`类有static代码块，加上try-catch之后放到线上，并放少量流量，查看日志发现几个问题：

1. XManager提供静态方法，它的属性都是静态变量；
2. 属性在static代码块中初始化；
3. 属性的初始化依赖由Spring加载的context。

服务启动运行main方法，main函数启动dubbo服务，能接收rpc服务，Spring加载bean，提供netty服务，由于主线程Spring初始化比较慢，导致rpc请求到达时，部分bean没有完成初始化context为空，从而导致static初始化失败，最后导致事故。

找到事故原因，处理起来就比较简单了，选了一种简单易行的办法，将XManager改为单例，并有Spring管理，依赖的属性交由Spring注入，这样就能解决这个问题。

#### 复现

在分析完成之后，需要复现异常，证实上面的部分推测，测试[源码](https://github.com/afredlyj/javaBaseDemo)。





