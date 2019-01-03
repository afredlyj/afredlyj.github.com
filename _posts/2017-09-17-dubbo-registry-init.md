---
layout: post
title: Dubbo Consumer端注册中心流程
category: program
---

Dubbo的注册中心提供服务注册、订阅和查找功能，本篇文章分析下Consumer端初始化的子流程——注册中心初始化及使用流程。

### Registry 初始化

Registry的初始化在`RegistryProtocol#refer`方法中完成：

```java

// 修改protocol 
url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);


// url的值为
// zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&logger=slf4j&organization=afred&owner=afred&pid=34305&refer=application%3Ddemo-provider%26check%3Dfalse%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.examples.test.HelloServiceAsync%26logger%3Dslf4j%26methods%3DsayHelloAsync%2CsayHello%26organization%3Dafred%26owner%3Dafred%26pid%3D34305%26side%3Dconsumer%26timeout%3D20000%26timestamp%3D1505005113647&timestamp=1505005113829, dubbo version: 2.0.0, current host: 127.0.0.1
// registryFactory也是生成的com.alibaba.dubbo.registry.RegistryFactory$Adpative类
Registry registry = registryFactory.getRegistry(url);

```

由于`registerFactory`对象也是SPI生成的动态代理，代码如下：

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

上文将protocol字段修改为registry对应的协议，就拿线上使用的zookeeper分析，调用`ZookeeperRegistryFactory#getRegistry`：

```java
// AbstractRegistryFactory.java
    public Registry getRegistry(URL url) {
    // 设置path和interface
    	url = url.setPath(RegistryService.class.getName())
    			.addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
    			.removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    	String key = url.toServiceString();
        // 锁定注册中心获取过程，保证注册中心单一实例
        LOCK.lock();
        try {
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            // 模版方法，由子类决定实现
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // 释放锁
            LOCK.unlock();
        }
    }

    protected abstract Registry createRegistry(URL url);

// ZookeeperRegistryFactory.java

public Registry createRegistry(URL url) {
    // Dubbo实现了类似Spring的ioc容器，zookeeperTransporter也是SPI机制生成的代理对象
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
    
```
类`ZookeeperRegistry`的类图如下：

![ZookeeperRegistry](https://afredlyj.github.io/assets/images/zookeeperregistry.png)

接下来分析该类的初始化：

```java
// AbstractRegistry.java
public AbstractRegistry(URL url) {
        setUrl(url);
        // 启动文件保存定时器
        syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
        String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getHost() + ".cache");
        File file = null;
        if (ConfigUtils.isNotEmpty(filename)) {
            file = new File(filename);
            if(! file.exists() && file.getParentFile() != null && ! file.getParentFile().exists()){
                if(! file.getParentFile().mkdirs()){
                    throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
                }
            }
        }
        this.file = file;
        loadProperties();
        notify(url.getBackupUrls());
}

// FailbackRegistry.java
public FailbackRegistry(URL url) {
        super(url);
        int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
        // 定时重连注册中心
        this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
            public void run() {
                // 检测并连接注册中心
                try {
                    retry();
                } catch (Throwable t) { // 防御性容错
                    logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
                }
            }
        }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
    }
    
// ZookeeperRegistry.java
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        super(url);
        if (url.isAnyHost()) {
    		throw new IllegalStateException("registry address == null");
    	}
        String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
        if (! group.startsWith(Constants.PATH_SEPARATOR)) {
            group = Constants.PATH_SEPARATOR + group;
        }
        this.root = group;
        // 实现可能是CuratorZookeeperTransporter或ZkclientZookeeperTransporter
        // 由于对ZkclientZookeeperTransporter不熟，这里只分析CuratorZookeeperTransporter
        zkClient = zookeeperTransporter.connect(url);
        // 添加监听器
        zkClient.addStateListener(new StateListener() {
            public void stateChanged(int state) {
            	if (state == RECONNECTED) {
	            	try {
						recover();
					} catch (Exception e) {
						logger.error(e.getMessage(), e);
					}
            	}
            }
        });
    }

```

`ZookeeperTransporter.connect`方法返回`ZookeeperClient`对象，通过该对象Dubbo实现对zookeeper注册中心的管理。`CuratorZookeeperClient`是该类的实现类。

```java
public class CuratorZookeeperTransporter implements ZookeeperTransporter {

	public ZookeeperClient connect(URL url) {
		return new CuratorZookeeperClient(url);
	}

}

//

public CuratorZookeeperClient(URL url) {
		super(url);
		// 完成curator client的初始化
		Builder builder = CuratorFrameworkFactory.builder()
				.connectString(url.getBackupAddress())
				.retryPolicy(new RetryNTimes(Integer.MAX_VALUE, 1000))
				.connectionTimeoutMs(url.getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_REGISTRY_CONNECT_TIMEOUT))
                .sessionTimeoutMs(url.getParameter(Constants.SESSION_TIMEOUT_KEY, Constants.DEFAULT_SESSION_TIMEOUT));
		String authority = url.getAuthority();
		if (authority != null && authority.length() > 0) {
			builder = builder.authorization("digest", authority.getBytes());
		}
		client = builder.build();
		// 增加监听器
		client.getConnectionStateListenable().addListener(
				new ConnectionStateListener() {
					public void stateChanged(CuratorFramework client,
							ConnectionState state) {
						if (state == ConnectionState.LOST) {
							CuratorZookeeperClient.this
									.stateChanged(StateListener.DISCONNECTED);
						} else if (state == ConnectionState.CONNECTED) {
							CuratorZookeeperClient.this
									.stateChanged(StateListener.CONNECTED);
						} else if (state == ConnectionState.RECONNECTED) {
							CuratorZookeeperClient.this
									.stateChanged(StateListener.RECONNECTED);
						}
					}
				});
		client.start();
	}
	
// 状态变更通知
protected void stateChanged(int state) {
		for (StateListener sessionListener : getSessionListeners()) {
			sessionListener.stateChanged(state);
		}
	}

```

到目前为止，我发现StateListener依然为空，那么StateListener列表从哪里添加的呢？

分析完ZookeeperClient的初始化，我们回到ZookeeperRegistry的构造函数中，发现有`zkClient.addStateListener`的操作，该方法会往`stateListeners`列表中添加匿名`StateListener`，所以`CuratorZookeeperClient`中添加的连接状态变更监听器接收到变更消息之后，会回调`ZookeeperRegistry`构造方法中的匿名类`stateChanged`逻辑，实现断线重连。

至此，`Registry`的初始化完成。

### Registry 注册

在初始化`RegistryDirectory`之后，Consumer端继续调用`registry.register`方法。由上文的类图可知，该方法由父类`FailbackRegistry`实现：

```java
@Override
    public void register(URL url) {
    // 调用父类AbstractRegistry的方法，往集合registered中添加url
        super.register(url);
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            // 向服务器端发送注册请求
            // 由子类实现，这里是ZookeeperRegistry
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;

            // 如果开启了启动时检测，则直接抛出异常
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && ! Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if(skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }

            // 将失败的注册请求记录到失败列表，定时重试
            failedRegistered.add(url);
        }
    }
    
// 子类实现doRegister
protected void doRegister(URL url) {
        try {
        	zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
    
// 创建zookeeper节点

public void create(String path, boolean ephemeral) {
		int i = path.lastIndexOf('/');
		if (i > 0) {
			create(path.substring(0, i), false);
		}
		if (ephemeral) {
			createEphemeral(path);
		} else {
			createPersistent(path);
		}
	}
```

注册流程的主要工作是在zookeeper上注册znode。接下来是订阅。

### RegistryDirectory 订阅

`RegistryDirectory#subscribe`方法完成对provider端的订阅。RegistryDirectory类图如下：

![image](https://github.com/afredlyj/afredlyj.github.com/blob/master/assets/images/registrydirectory.png)


```java
// 初始化
// RegistryProtocol.java  
 RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
 // 即为上文中的Registry
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        
// 订阅
directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
                Constants.PROVIDERS_CATEGORY 
                + "," + Constants.CONFIGURATORS_CATEGORY 
                + "," + Constants.ROUTERS_CATEGORY));
                
// 
public void subscribe(URL url) {
        setConsumerUrl(url);
        // 调用Registry订阅，并监听器为RegistryDirectory，notify方法逻辑稍后分析
        // subscribe最后调用ZookeeperRegistry的doSubscribe方法
        registry.subscribe(url, this);
    }
    
// FailbackRegistry.java
@Override
    public void subscribe(URL url, NotifyListener listener) {

        super.subscribe(url, listener);
        removeFailedSubscribed(url, listener);
        try {
            // 向服务器端发送订阅请求
            // 模版方法，由子类实现，这里我分析ZookeeperRegistry，listener是RegistryDirectory对象，在subscribe方法中传入
            doSubscribe(url, listener);
        } catch (Exception e) {
            Throwable t = e;

            List<URL> urls = getCacheUrls(url);
            if (urls != null && urls.size() > 0) {
                notify(url, listener, urls);
                logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
            } else {
                // 如果开启了启动时检测，则直接抛出异常
                boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                        && url.getParameter(Constants.CHECK_KEY, true);
                boolean skipFailback = t instanceof SkipFailbackWrapperException;
                if (check || skipFailback) {
                    if(skipFailback) {
                        t = t.getCause();
                    }
                    throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
                } else {
                    logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
                }
            }

            // 将失败的订阅请求记录到失败列表，定时重试
            addFailedSubscribed(url, listener);
        }
    }

```

跟踪doSubscribe方法，最终会调用`RegistryDirectory#notify`方法，接受zookeeper节点变更通知，到此为止，都还没有看到Invoker相关的内容，那么接下来就能步入正题：

```java
// RegistryDirectory.java
public synchronized void notify(List<URL> urls) {
        List<URL> invokerUrls = new ArrayList<URL>();
        List<URL> routerUrls = new ArrayList<URL>();
        List<URL> configuratorUrls = new ArrayList<URL>();
        for (URL url : urls) {
            String protocol = url.getProtocol();
            String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            if (Constants.ROUTERS_CATEGORY.equals(category) 
                    || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                routerUrls.add(url);
            } else if (Constants.CONFIGURATORS_CATEGORY.equals(category) 
                    || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                configuratorUrls.add(url);
            } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                invokerUrls.add(url);
            } else {
                logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
            }
        }
        // configurators 
        if (configuratorUrls != null && configuratorUrls.size() >0 ){
            this.configurators = toConfigurators(configuratorUrls);
        }
        // routers
        if (routerUrls != null && routerUrls.size() >0 ){
            List<Router> routers = toRouters(routerUrls);
            if(routers != null){ // null - do nothing
                setRouters(routers);
            }
        }
        List<Configurator> localConfigurators = this.configurators; // local reference
        // 合并override参数
        this.overrideDirectoryUrl = directoryUrl;
        if (localConfigurators != null && localConfigurators.size() > 0) {
            for (Configurator configurator : localConfigurators) {
                this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
            }
        }
        // providers
        refreshInvoker(invokerUrls);
    }

    /**
     * 根据invokerURL列表转换为invoker列表。转换规则如下：
     * 1.如果url已经被转换为invoker，则不在重新引用，直接从缓存中获取，注意如果url中任何一个参数变更也会重新引用
     * 2.如果传入的invoker列表不为空，则表示最新的invoker列表
     * 3.如果传入的invokerUrl列表是空，则表示只是下发的override规则或route规则，需要重新交叉对比，决定是否需要重新引用。
     * @param invokerUrls 传入的参数不能为null
     */
    private void refreshInvoker(List<URL> invokerUrls){

        logger.info("更新 invoker : " + invokerUrls);

        if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
                && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
            this.forbidden = true; // 禁止访问
            this.methodInvokerMap = null; // 置空列表
            destroyAllInvokers(); // 关闭所有Invoker
        } else {
            this.forbidden = false; // 允许访问
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
            if (invokerUrls.size() == 0 && this.cachedInvokerUrls != null){
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<URL>();
                this.cachedInvokerUrls.addAll(invokerUrls);//缓存invokerUrls列表，便于交叉对比
            }
            if (invokerUrls.size() ==0 ){
            	return;
            }
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls) ;// 将URL列表转成Invoker列表
            Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap); // 换方法名映射Invoker列表
            // state change
            //如果计算错误，则不进行处理.
            if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0 ){
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :"+invokerUrls.size() + ", invoker.size :0. urls :"+invokerUrls.toString()));
                return ;
            }
            this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
            this.urlInvokerMap = newUrlInvokerMap;
            try{
                destroyUnusedInvokers(oldUrlInvokerMap,newUrlInvokerMap); // 关闭未使用的Invoker
            }catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }


    /**
     * 将urls转成invokers,如果url已经被refer过，不再重新引用。
     * 
     * @param urls
     * @param overrides
     * @param query
     * @return invokers
     */
    private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
        Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<String, Invoker<T>>();
        if(urls == null || urls.size() == 0){
            return newUrlInvokerMap;
        }
        Set<String> keys = new HashSet<String>();
        String queryProtocols = this.queryMap.get(Constants.PROTOCOL_KEY);
        for (URL providerUrl : urls) {
        	//如果reference端配置了protocol，则只选择匹配的protocol
        	if (queryProtocols != null && queryProtocols.length() >0) {
        		boolean accept = false;
        		String[] acceptProtocols = queryProtocols.split(",");
        		for (String acceptProtocol : acceptProtocols) {
        			if (providerUrl.getProtocol().equals(acceptProtocol)) {
        				accept = true;
        				break;
        			}
        		}
        		if (!accept) {
        			continue;
        		}
        	}
            if (Constants.EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
                continue;
            }
            if (! ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
                logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() + " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost() 
                        + ", supported protocol: "+ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
                continue;
            }
            URL url = mergeUrl(providerUrl);
            
            String key = url.toFullString(); // URL参数是排序的
            if (keys.contains(key)) { // 重复URL
                continue;
            }
            keys.add(key);
            // 缓存key为没有合并消费端参数的URL，不管消费端如何合并参数，如果服务端URL发生变化，则重新refer
            Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
            Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
            if (invoker == null) { // 缓存中没有，重新refer
                try {
                	boolean enabled = true;
                	if (url.hasParameter(Constants.DISABLED_KEY)) {
                		enabled = ! url.getParameter(Constants.DISABLED_KEY, false);
                	} else {
                		enabled = url.getParameter(Constants.ENABLED_KEY, true);
                	}
                	if (enabled) {
                	// 创建Invoker 代理
                		invoker = new InvokerDelegete<T>(protocol.refer(serviceType, url), url, providerUrl);
                	}
                } catch (Throwable t) {
                    logger.error("Failed to refer invoker for interface:"+serviceType+",url:("+url+")" + t.getMessage(), t);
                }
                if (invoker != null) { // 将新的引用放入缓存
                    newUrlInvokerMap.put(key, invoker);
                }
            }else {
                newUrlInvokerMap.put(key, invoker);
            }
        }
        keys.clear();
        return newUrlInvokerMap;
    }

```

分析了这么多，注册中心的大概流程就是这样，但是对于Consumer端异步优化，修改interface字段放在哪里合适，还没有定论。

接下来继续分析`Invoker`的创建和初始化流程，希望能找到解决方案。