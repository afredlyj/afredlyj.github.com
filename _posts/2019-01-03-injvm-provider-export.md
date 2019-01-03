---
layout: post
title: Dubbo Provider 本地暴露流程
category: program
---

### 类图

以Spring  XML + Dubbo为例，在Provider端启动时，会初始化ServiceBean：

![图片无法显示](https://github.com/afredlyj/afredlyj.github.com/tree/master/assets/images/dubbo-provider-spring-schema.png "异常信息") 

本章主要关注ServiceBean的初始化，ServiceBean实现了InitializingBean和DisposableBean接口，了解Spring机制的都知道这两个类的作用，顺藤摸瓜即可理解Dubbo 服务暴露的入口了。

![图片无法显示](https://github.com/afredlyj/afredlyj.github.com/tree/master/assets/images/dubbo-servicebean.png "异常信息") 



### JavassistProxyFactory生成Invoker

在分析之前流程之前，借用Dubbo官网的描述说明Invoker在整个调用流程中的重要性：

> 从Dubbo 官网可知，由于 Invoker 是 Dubbo 领域模型中非常重要的一个概念，很多设计思路都是向它靠拢。这就使得 Invoker 渗透在整个实现代码里，对于刚开始接触 Dubbo 的人，确实容易给搞混了。 下面我们用一个精简的图来说明最重要的两种 Invoker：服务提供 Invoker 和服务消费 Invoker：

![图片无法显示](https://github.com/afredlyj/afredlyj.github.com/tree/master/assets/images/dubbo-invoker.png "异常信息") 

略去`ServiceBean`的初始化流程，分析到Provider端Invoder的生成流程。在分析Dubbo的流程时，一般都是调试Dubbo的测试代码，逐步debug，在这里也是一样，启动`com.alibaba.dubbo.examples.version.VersionProvider`，跟踪到生成Invoker代码如下：

```java
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

```

根据上述代码，会先根据代理对象生成装饰类，具体代码如下：

```java

public static Wrapper getWrapper(Class<?> c) {
        while (ClassGenerator.isDynamicClass(c)) // can not wrapper on dynamic class.
            c = c.getSuperclass();

        if (c == Object.class)
            return OBJECT_WRAPPER;

        //  优先从缓存中取对应的Wrapper对象，如果不存在，动态生成Wrapper class，并创建Wrapper对象
        Wrapper ret = WRAPPER_MAP.get(c);
        if (ret == null) {
            ret = makeWrapper(c);
            WRAPPER_MAP.put(c, ret);
        }
        return ret;
    }
```

创建Wrapper类的过程，由Dubbo封装`javassist`完成，具体流程这里不分析。因为Dubbo自动生成的类太多，为了方便理解，所以在调试时使用ClassDump把Wrapper类打印出来如下，方便分析流程：

```java
package com.alibaba.dubbo.common.bytecode;

import com.alibaba.dubbo.examples.version.impl.VersionServiceImpl;
import java.lang.reflect.InvocationTargetException;
import java.util.Map;

public class Wrapper1
  extends Wrapper
  implements ClassGenerator.DC
{
  public static String[] pns;
  public static Map pts;
  public static String[] mns;
  public static String[] dmns;
  public static Class[] mts0;
  
  public String[] getPropertyNames()
  {
    return pns;
  }
  
  public String[] getMethodNames()
  {
    return mns;
  }
  
  public boolean hasProperty(String paramString)
  {
    return pts.containsKey(paramString);
  }
  
  public Class getPropertyType(String paramString)
  {
    return (Class)pts.get(paramString);
  }
  
  public String[] getDeclaredMethodNames()
  {
    return dmns;
  }
  
  public Object invokeMethod(Object paramObject, String paramString, Class[] paramArrayOfClass, Object[] paramArrayOfObject)
    throws InvocationTargetException
  {
    VersionServiceImpl localVersionServiceImpl;
    try
    {
      localVersionServiceImpl = (VersionServiceImpl)paramObject;
    }
    catch (Throwable localThrowable1)
    {
      throw new IllegalArgumentException(localThrowable1);
    }
    try
    {
      if ((!"sayHello".equals(paramString)) || (paramArrayOfClass.length == 1)) {
        return localVersionServiceImpl.sayHello((String)paramArrayOfObject[0]);
      }
    }
    catch (Throwable localThrowable2)
    {
      throw new InvocationTargetException(localThrowable2);
    }
    throw new NoSuchMethodException("Not found method \"" + paramString + "\" in class com.alibaba.dubbo.examples.version.impl.VersionServiceImpl.");
  }
  
  public Object getPropertyValue(Object paramObject, String paramString)
  {
    try
    {
      VersionServiceImpl localVersionServiceImpl = (VersionServiceImpl)paramObject;
    }
    catch (Throwable localThrowable)
    {
      throw new IllegalArgumentException(localThrowable);
    }
    throw new NoSuchPropertyException("Not found property \"" + paramString + "\" filed or setter method in class com.alibaba.dubbo.examples.version.impl.VersionServiceImpl.");
  }
  
  public void setPropertyValue(Object paramObject1, String paramString, Object paramObject2)
  {
    try
    {
      VersionServiceImpl localVersionServiceImpl = (VersionServiceImpl)paramObject1;
    }
    catch (Throwable localThrowable)
    {
      throw new IllegalArgumentException(localThrowable);
    }
    throw new NoSuchPropertyException("Not found property \"" + paramString + "\" filed or setter method in class com.alibaba.dubbo.examples.version.impl.VersionServiceImpl.");
  }
}

```

返回`AbstractProxyInvoker`的一个实现类对象，跟踪该对象的`invoke`方法：

```java

	@Override
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

最终会调用匿名类的`doInvoke`方法，而`doInvoke`则会调用`Wrapper#invokeMethod`方法，该方法最终调用代理对象的指定方法，这就是Invoker之后实际的调用流程。

疑问：Invoker之前的调用流程是什么样子的？


### 本地暴露流程

分析完Invoker对象的生成和执行流程，回到本地暴露方法`exportLocal`方法：

```java
    private void exportLocal(URL url) {
    // 如果当前协议不是本地暴露，则进行本地暴露
        if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
            URL local = URL.valueOf(url.toFullString())
                    // 改写Protocol，改写前值为registry，改写后为injvm
                    .setProtocol(Constants.LOCAL_PROTOCOL)
                    .setHost(LOCALHOST)
                    .setPort(0);
            
            // 暂时没理解这段代码的意义
            ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
            
            // 核心代码
            // proxyFactory.getInvoker(ref, (Class) interfaceClass, local) 生成Invoker对象的url protocol为injvm
            // exporter 为ListenerExporterWrapper的对象
            Exporter<?> exporter = protocol.export(
                    proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
            // 添加到列表中
            exporters.add(exporter);
            logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
        }
    }

```

上述核心代码有两个关键变量`protocol`和`proxyFactory`，在`ServiceConfig`类中引用的`proxyFactory`代码如下：

```java  
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```  

根据Dubbo SPI机制，`proxyFactory`实际上是`StubProxyFactoryWrapper`对象，默认情况下是对`JavassistProxyFactory`对象的包装，实现如下：


```java
	@Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException {
    // proxyFactory 为 JavassistProxyFactory 对象
        return proxyFactory.getInvoker(proxy, type, url);
    }
```

同样，`Protocol`也默认有几个Wrapper类，分别是：`ProtocolListenerWrapper`、`ProtocolFilterWrapper`和`QosProtocolWrapper`：


![图片无法显示](https://github.com/afredlyj/afredlyj.github.com/tree/master/assets/images/injvm_protocol.png "异常信息")  

`JavassistProxyFactory#getInvoker`流程上一章节已经分析，不再复述，综上所述，Dubbo Provider 本地暴露的流程如下：

1. `ServiceBean#afterpropertiesSet`初始化，调用`export`方法；
2. `export`方法遍历所有的`protocols`，调用`doExportUrlsFor1Protocol`方法；
3. `doExportUrlsFor1Protocol`会判断`scope`的值；
4. 如果`scope`不是`remote`，并且`protocol`的值不为`injvm`，则会暴露本地服务；
5. 本地服务的protocol为`invjm`，host为`127.0.0.1`，port为`0`；
6. 根据Dubbo SPI，会分别调用`StubProxyFactoryWrapper`和`JavassistProxyFactory`的`getInvoker`方法，构建Invoker对象——一个`AbstractProxyInvoker`匿名实现类的对象，Invoker对象封装了对实际目标对象的调用；
7. protocol根据invoker对象生成Exporter对象；
8. 将生成的Exporter对象添加到exporters中，流程结束。