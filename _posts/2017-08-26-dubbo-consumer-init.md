---
layout: post
title: Dubbo consumer 端启动流程
category: program
---

Dubbo对Java `SPI`机制进行扩展，并结合`URL`能对整个流程方便扩展。
### SPI扩展

SPI定义如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {

    /**
     * 缺省扩展点名。
     */
	String value() default "";

}
```
Dubbo对SPI机制做了扩展，只有在打了`@SPI`注解的接口才会查找扩展点的实现，查找过程是依次遍历文件目录：

```java
META-INF/dubbo/internal/
META-INF/dubbo/
META-INFO/services
```
比如`Protocol`接口的扩展点`DubboProtocol`配置文件com.alibaba.dubbo.rpc.protocol.dubbo.Protocol内容如下：
```
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
```

Dubbo中SPI功能扩展的核心类是`ExtensionLoader`，使用方法如下：

```java

// 加载Protocol 的默认实现
private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

```

#### 加载扩展点
核心流程在方法`loadExtensionClasses`中：

```java
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        // 加载默认实现
        if(defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if(value != null && (value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if(names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if(names.length == 1) cachedDefaultName = names[0];
            }
        }
        
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }

```

`loadFile`方法负责加载指定文件，文件内容如上文，以key=value的形式存储，其中key为扩展点名称，value为具体实现类，loadFile加载流程如下：
1. 判断实现类是否有注解`Adaptive`，如果有，则将此类作为适配器类缓存在字段`cachedAdaptiveClass`中；  
2. 否则，判断实现类是否存在入参为扩展点接口的构造器，判断方法如下：  
```java
clazz.getConstructor(type);
```  
如果有，则将该类作为装饰类，加入到cachedWrapperClasses中；  
3. 否则，说明是扩展点的具体实现对象，查询实现类上是否有注解`Activate`，有则缓存到cachedActivates中，同时将实现类缓存到extensionClasses中；  
4. 如果`cachedAdaptiveClass`为空，则调用下面的方法，生成适配器类：		
```java
private Class<?> createAdaptiveExtensionClass() {
// 生成代码，不过有生成条件
        String code = createAdaptiveExtensionClassCode();
        ClassLoader classLoader = findClassLoader();
       // 获取Complier 扩展点的具体实现
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```

适配器类的作用是通过`url.getProtocol`方法获取到`extName`，根据这个参数选择具体的扩展点实现类。生成适配器类的条件如下：

1. 接口方法中必须至少要有一个有`Adaptive`注解；
2. 标有`Adaptive`注解的方法，必须要有URL类型参数，或者参数中存在getURL方法。

具体的判断逻辑可以详细阅读`ExtensionLoader#createAdaptiveExtensionClassCode`。以Protocol生成的适配器代码为例（createAdaptiveExtensionClassCode方法返回的是String，可以直接将生成的代码debug出来）：

```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;


public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {

   // 没有Adaptive的方法，如果调用，会抛异常
    public void destroy() {
        throw new UnsupportedOperationException(
            "method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException(
            "method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

	// Protocol接口在export上有Adaptive注解
    public com.alibaba.dubbo.rpc.Exporter export(
        com.alibaba.dubbo.rpc.Invoker arg0)
        throws com.alibaba.dubbo.rpc.Invoker {
        if (arg0 == null) {
            throw new IllegalArgumentException(
                "com.alibaba.dubbo.rpc.Invoker argument == null");
        }

        if (arg0.getUrl() == null) {
            throw new IllegalArgumentException(
                "com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        }

        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = ((url.getProtocol() == null) ? "dubbo"
                                                      : url.getProtocol());

        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" +
                url.toString() + ") use keys([protocol])");
        }

        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
                                                                                                   .getExtension(extName);

        return extension.export(arg0);
    }

// Protocol接口在refer上有Adaptive注解
    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0,
        com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
        if (arg1 == null) {
            throw new IllegalArgumentException("url == null");
        }

        com.alibaba.dubbo.common.URL url = arg1;
        // 获取extName
        String extName = ((url.getProtocol() == null) ? "dubbo"
                                                      : url.getProtocol());

        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" +
                url.toString() + ") use keys([protocol])");
        }

		// 根据extName获取扩展点的具体实现
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
                                                                                                   .getExtension(extName);

        return extension.refer(arg0, arg1);
    }
}

```

生成适配器代码之后，通过Dubbo统一封装的`Compiler`类，编译成Class对象，Dubbo提供两种代码编译的方式，分别是jdk和javassit，那么问题来了，`Compiler`如果也是通过以上方式，生成java代码，然后编译，整个流程似乎进入了死胡同。

事实当然不是这样，如果仔细研究上文中`loadFile`的代码就会发现，`Adaptive`注解打在具体实现类和打在接口上是有区别的：  
1. 如果注解打在实现类上，则会设置`cachedAdaptiveClass`参数，`getExtensionClasses`执行完成之后，判断该参数非空，则直接将该Class对象返回；  
2. 否则，调用`createAdaptiveExtensionClass`方法，走上面的生成代码流程。

而`Compiler`的定义，我们可以看看：

```java
@SPI("javassist")
public interface Compiler {

	// 接口方法上并没有Adaptive注解
	/**
	 * Compile java source code.
	 * 
	 * @param code Java source code
	 * @param classLoader TODO
	 * @return Compiled class
	 */
	Class<?> compile(String code, ClassLoader classLoader);

}
``` 

配置文件内容如下：  

```java  
adaptive=com.alibaba.dubbo.common.compiler.support.AdaptiveCompiler
jdk=com.alibaba.dubbo.common.compiler.support.JdkCompiler
javassist=com.alibaba.dubbo.common.compiler.support.JavassistCompiler  
```

在实现类`AdaptiveCompiler `上有注解：

```java
@Adaptive
public class AdaptiveCompiler implements Compiler {

    private static volatile String DEFAULT_COMPILER;

    public static void setDefaultCompiler(String compiler) {
        DEFAULT_COMPILER = compiler;
    }

    public Class<?> compile(String code, ClassLoader classLoader) {
        Compiler compiler;
        ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
        String name = DEFAULT_COMPILER; // copy reference
        if (name != null && name.length() > 0) {
            compiler = loader.getExtension(name);
        } else {
            compiler = loader.getDefaultExtension();
        }
        return compiler.compile(code, classLoader);
    }

}

```

所以Compiler类，获取的扩展点实现类为`AdaptiveCompiler`。
扩展点还会返回包装类：

1. 在`ExtensionLoader.loadFile`加载扩展点配置文件时，如果加载到的类有接口类型为参数的构造函数，则该类就是装饰类，缓存到`cachedWrapperClasses`中：  
```java
 clazz.getConstructor(type);
 Set<Class<?>> wrappers = cachedWrapperClasses;
if (wrappers == null) {
	cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();                                                        wrappers = cachedWrapperClasses;
 }
wrappers.add(clazz);
```
比如`ProtocolFilterWrapper`就是`Protocol`的装饰类：  
```java
  public ProtocolFilterWrapper(Protocol protocol){
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
```
2. 装饰器类的实例化，在`ExtensionLoader#createExtension`方法中，如果当前扩展点key存在装饰器，则在创建并初始化扩展点对象之后（实现类似Spring 的IOC功能），循环遍历`cachedWrapperClasses`并初始化返回，也就是说，对于有装饰器的扩展点，返回的是装饰器对象。

```java
@SuppressWarnings("unchecked")
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && wrapperClasses.size() > 0) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance))c;
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }

```

以上就是Dubbo扩展点实现的技术细节，`Protocol`、`Cluster`、`ProxyFactory`等扩展点的实现机制都是这样。

还有`RegistryFactory`扩展类的代码，下文分析流程需要用到，所以一并贴出来：

```java


public class RegistryFactory$Adpative implements com.alibaba.dubbo.registry.RegistryFactory {
    public com.alibaba.dubbo.registry.Registry getRegistry(
        com.alibaba.dubbo.common.URL arg0) {
        if (arg0 == null) {
            throw new IllegalArgumentException("url == null");
        }

        com.alibaba.dubbo.common.URL url = arg0;
        String extName = ((url.getProtocol() == null) ? "dubbo"
                                                      : url.getProtocol());

        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.registry.RegistryFactory) name from url(" +
                url.toString() + ") use keys([protocol])");
        }

        com.alibaba.dubbo.registry.RegistryFactory extension = (com.alibaba.dubbo.registry.RegistryFactory) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.registry.RegistryFactory.class)
                                                                                                                           .getExtension(extName);

        return extension.getRegistry(arg0);
    }
}

```


### ReferenceBean的初始化

Dubbo 支持Spring无缝集成，xml schema解析类如下：

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

	static {
		Version.checkDuplicate(DubboNamespaceHandler.class);
	}

	public void init() {
	    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }

}
```

consumer端启动流程我们关注下`ReferenceBean`，该类的继承关系如下：

```java
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {

}

```

在Spring中，有两种bean，分别是普通bean和factory bean，factory bean返回的并不是本身，而是`getObject`方法返回的对象，而`ReferenceBean`就是一个factory bean, 他返回的是一个consumer端代理，接下来主要看`getObject`方法的流程：

```java
{
        if (initialized) {
            return;
        }
        initialized = true;
        if (interfaceName == null || interfaceName.length() == 0) {
            throw new IllegalStateException("<dubbo:reference interface=\"\" /> interface not allow null!");
        }
        // 获取消费者全局配置
        checkDefault();
        appendProperties(this);
        if (getGeneric() == null && getConsumer() != null) {
            setGeneric(getConsumer().getGeneric());
        }
        if (ProtocolUtils.isGeneric(getGeneric())) {
            interfaceClass = GenericService.class;
        } else {
            try {
                interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                        .getContextClassLoader());
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            checkInterfaceAndMethods(interfaceClass, methods);
        }
        String resolve = System.getProperty(interfaceName);
        String resolveFile = null;
        if (resolve == null || resolve.length() == 0) {
            resolveFile = System.getProperty("dubbo.resolve.file");
            if (resolveFile == null || resolveFile.length() == 0) {
                File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
                if (userResolveFile.exists()) {
                    resolveFile = userResolveFile.getAbsolutePath();
                }
            }
            if (resolveFile != null && resolveFile.length() > 0) {
                Properties properties = new Properties();
                FileInputStream fis = null;
                try {
                    fis = new FileInputStream(new File(resolveFile));
                    properties.load(fis);
                } catch (IOException e) {
                    throw new IllegalStateException("Unload " + resolveFile + ", cause: " + e.getMessage(), e);
                } finally {
                    try {
                        if (null != fis) fis.close();
                    } catch (IOException e) {
                        logger.warn(e.getMessage(), e);
                    }
                }
                resolve = properties.getProperty(interfaceName);
            }
        }
        if (resolve != null && resolve.length() > 0) {
            url = resolve;
            if (logger.isWarnEnabled()) {
                if (resolveFile != null && resolveFile.length() > 0) {
                    logger.warn("Using default dubbo resolve file " + resolveFile + " replace " + interfaceName + "" + resolve + " to p2p invoke remote service.");
                } else {
                    logger.warn("Using -D" + interfaceName + "=" + resolve + " to p2p invoke remote service.");
                }
            }
        }
        if (consumer != null) {
            if (application == null) {
                application = consumer.getApplication();
            }
            if (module == null) {
                module = consumer.getModule();
            }
            if (registries == null) {
                registries = consumer.getRegistries();
            }
            if (monitor == null) {
                monitor = consumer.getMonitor();
            }
        }
        if (module != null) {
            if (registries == null) {
                registries = module.getRegistries();
            }
            if (monitor == null) {
                monitor = module.getMonitor();
            }
        }
        if (application != null) {
            if (registries == null) {
                registries = application.getRegistries();
            }
            if (monitor == null) {
                monitor = application.getMonitor();
            }
        }
        checkApplication();
        checkStubAndMock(interfaceClass);
        Map<String, String> map = new HashMap<String, String>();
        Map<Object, Object> attributes = new HashMap<Object, Object>();
        map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
        map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
        map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
        if (ConfigUtils.getPid() > 0) {
            map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
        }
        if (!isGeneric()) {
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put("revision", revision);
            }

            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if (methods.length == 0) {
                logger.warn("NO method found in service interface " + interfaceClass.getName());
                map.put("methods", Constants.ANY_VALUE);
            } else {
                map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
            }
        }
        map.put(Constants.INTERFACE_KEY, interfaceName);
        appendParameters(map, application);
        appendParameters(map, module);
        appendParameters(map, consumer, Constants.DEFAULT_KEY);
        appendParameters(map, this);
        String prifix = StringUtils.getServiceKey(map);
        if (methods != null && methods.size() > 0) {
            for (MethodConfig method : methods) {
                appendParameters(map, method, method.getName());
                String retryKey = method.getName() + ".retry";
                if (map.containsKey(retryKey)) {
                    String retryValue = map.remove(retryKey);
                    if ("false".equals(retryValue)) {
                        map.put(method.getName() + ".retries", "0");
                    }
                }
                appendAttributes(attributes, method, prifix + "." + method.getName());
                checkAndConvertImplicitConfig(method, map, attributes);
            }
        }
        //attributes通过系统context进行存储.
        StaticContext.getSystemContext().putAll(attributes);
        // 关键代码，通过map创建代理对象
        ref = createProxy(map);
    }
```

上述代码做了大量的初始化工作，最后将初始化配置放到map中，通过map传递参数，生成代理对象，初始化流程这里不详细追踪，接下来看生成代理的流程：

```java
@SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
    private T createProxy(Map<String, String> map) {
        URL tmpUrl = new URL("temp", "localhost", 0, map);
        // 是否走本地引用
        final boolean isJvmRefer;
        if (isInjvm() == null) {
            if (url != null && url.length() > 0) { //指定URL的情况下，不做本地引用
                isJvmRefer = false;
            } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
                //默认情况下如果本地有服务暴露，则引用本地服务.
                isJvmRefer = true;
            } else {
                isJvmRefer = false;
            }
        } else {
            isJvmRefer = isInjvm().booleanValue();
        }

        if (isJvmRefer) {
            URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
            invoker = refprotocol.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            if (url != null && url.length() > 0) { // 用户指定URL，指定的URL可能是对点对直连地址，也可能是注册中心URL
                String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (url.getPath() == null || url.getPath().length() == 0) {
                            url = url.setPath(interfaceName);
                        }
                        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // 通过注册中心配置拼装URL
                List<URL> us = loadRegistries(false);
                if (us != null && us.size() > 0) {
                    for (URL u : us) {
                        URL monitorUrl = loadMonitor(u);
                        if (monitorUrl != null) {
                            map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                        urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    }
                }
                if (urls == null || urls.size() == 0) {
                    throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }
			  // 生成的适配类 class com.alibaba.dubbo.rpc.Protocol$Adpative
            logger.info("refprotocol : " + refprotocol.getClass());


            if (urls.size() == 1) {
            // url : 
            // registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&organization=afred&owner=afred&pid=4097&refer=application%3Ddemo-provider%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.examples.test.HelloServiceAsync%26methods%3DsayHelloAsync%2CsayHello%26organization%3Dafred%26owner%3Dafred%26pid%3D4097%26side%3Dconsumer%26timeout%3D1000%26timestamp%3D1494040307587&registry=zookeeper&timestamp=1494040307676, dubbo version: 2.0.0, current host: 127.0.0.1
            // protocol : registry
            
                logger.info("url : " + urls.get(0));
			// 根据上文Protocol$Adpative的源码，refer方法根据urls.get(0)的配置，会返回RegistryProtocol.refer返回的对象，即MockClusterInvoker
			// 将consumer端注册到注册中心
			// 订阅provider服务变更信息
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // 用了最后一个registry url
                    }
                }
                if (registryURL != null) { // 有 注册中心协议的URL
                    // 对有注册中心的Cluster 只用 AvailableCluster
                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                } else { // 不是 注册中心的URL
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        }

        logger.info("invoker : " + invoker.getClass());

        Boolean c = check;
        if (c == null && consumer != null) {
            c = consumer.isCheck();
        }
        if (c == null) {
            c = true; // default true
        }
        if (c && !invoker.isAvailable()) {
            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        // 创建服务代理
        return (T) proxyFactory.getProxy(invoker);
    }
```

看一下`refprotocol.refer`的流程，即`RegistryProtocol#refer`：

```java
@SuppressWarnings("unchecked")
	public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        
        // 根据上文ReferenceConfig中的urls.get(0)的内容，参数url的值为 registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&logger=slf4j&organization=afred&owner=afred&pid=25409&refer=application%3Ddemo-provider%26check%3Dfalse%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.examples.test.HelloServiceAsync%26logger%3Dslf4j%26methods%3DsayHelloAsync%2CsayHello%26organization%3Dafred%26owner%3Dafred%26pid%3D25409%26side%3Dconsumer%26timeout%3D20000%26timestamp%3D1503739193056&registry=zookeeper&timestamp=1503739193270
        // registryFactory的类型是RegistryFactory$Adpative，代码上节已经贴出来了
        // 所以这里最终会调用ZookeeperRegistryFactory#getRegistry方法生成ZookeeperRegistry对象
        
        Registry registry = registryFactory.getRegistry(url);
        logger.info("register factory : " + registryFactory.getClass());
        logger.info("register : " + registry.getClass());
        if (RegistryService.class.equals(type)) {
        	return proxyFactory.getInvoker((T) registry, type, url);
        }

        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
        String group = qs.get(Constants.GROUP_KEY);
        if (group != null && group.length() > 0 ) {
            if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                    || "*".equals( group ) ) {
                return doRefer( getMergeableCluster(), registry, type, url );
            }
        }
        return doRefer(cluster, registry, type, url);
    }
    
    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
        if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            // 将consumer端注册到注册中心registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                    Constants.CHECK_KEY, String.valueOf(false)));
        }
        
        // 订阅变更通知 directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
                Constants.PROVIDERS_CATEGORY 
                + "," + Constants.CONFIGURATORS_CATEGORY 
                + "," + Constants.ROUTERS_CATEGORY));
        // cluster 也是扩展点类  Cluster$Adpative
        // 默认会调用FailoverCluster.join方法，生成FailoverClusterInvoker对象
        // 存疑：实际返回的是MockClusterInvoker对象
        // 解答：由于定义了Cluster接口的装饰器类MockClusterWrapper，所以cluster.join实际上调用的是MockClusterInvoker#join方法，该方法返回MockClusterInvoker对象，MockClusterInvoker对象封装了FailoverCluster.join返回的FailoverClusterInvoker对象。
        return cluster.join(directory);
    }
```

`invoker`是dubbo中一个很重要的组件，封装了Provider地址及Service接口信息。生成invoker后，接下来看看怎样生成代理：

```java
// 注意，在我分析的这篇文章中，到目前为止invoker是MockClusterInvoker对象。
        return (T) proxyFactory.getProxy(invoker);
```

这里的`proxyFactory`为com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory，顺着该类的源码，就能找到代理生成的流程，这里不再详述。

综上所诉，Consumer端生成代理的流程总结如下：
 1. spring加载时，解析Dubbo定义的命名空间，生成配置相关的对象；  
 2. consumer对应的直接配置为ReferenceConfig， reference的config加载完成后，分别使用ConsumerConfig、ApplicationConfig、ModuleConfig、RegistryConfig、MonitorConfig等的默认值来初始化ReferenceConfig，ReferenceConfig是FactoryBean，实际返回getObject方法返回的对象；  
 3. 获取注册中心配置，根据配置连接注册中心，并注册和订阅url变动提醒；  
 4. 根据URL的配置生成invoker，该对象是Dubbo中的可执行体，方法的调用实际都是通过他来实现；  
 5. 根据接口生成代理对象，创建时传入InvokerInvocationHandler，该handler封装了第4步生成的invoker，不管是`JavassistProxyFactory`还是`JdkProxyFactory`生成的代理对象，最终都是通过`InvokerInvocationHandler`调用`invoker.invoke`方法。  
 

在整个初始化过程中，大量运用`SPI`机制，并通过URL总线生成并返回目标对象，加大对源码的分析难度，里面还有一些细节需要再找时间梳理。