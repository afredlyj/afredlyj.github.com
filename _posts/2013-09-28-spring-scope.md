---
layout: post
title: Spring scope引发的问题
category: bug 
---

Spring常用的bean scope有singleton和prototype，当然还有Web相关的scope，刚刚编码的时候看Spring初始化日志，发现尽管只有一次Http请求，但是有一个bean总是被注入两次，部分代码如下：  

~~~~
// MerQueryInfoBusiness.java  
  public class MerQueryInfoBusiness {  
    @Resource(name = "cacheManager")  
	 private CacheManager cacheManager;  
  }   

// CacheManager.java  
  public class CacheManager {  
	   private AbstractCache cache;  
  }  

// AbstractCache.java  
  public abstract class AbstractCache  
    implements ICache  
  {  
    private ICache next;  

  public ICache getNext()  
  {  
    return this.next;  
  }  

  public void setNext(ICache next) {  
    this.next = next;  
  }  
}  
~~~~  

AbstractCache的实现类有RedisCache和DBCache，其中RedisCache是第一级缓存，DBCcache是RedisCache的下一级缓存，Http请求来的时候，会进入业务处理类MerQueryInfoBusiness，这个业务类会将数据插入数据库和缓存，Spring的有关配置文件:

~~~~
	<bean id="redisCache" class="com.fruit.cache.RedisCache" init-method="init" scope="prototype">  
		<property name="next" ref="dbCache"></property>  
	</bean>  
   	<bean id="cacheManager" class="com.fruit.cache.CacheManager" scope="prototype">  
		<property name="cache" ref="redisCache"></property>  
		<property name="needCache" value="true"></property>  
	<property name="needDB" value="true"></property>  
	</bean>  
    	<bean id="merQueryInfoService" class="com.fruit.business.MerQueryInfoBusiness"  
		scope="prototype">  
 	<property name="cacheManager" ref="cacheManager" />  
	</bean>  
~~~~

从上面的代码可以看出，整个代码结构有点乱，注入bean既用了Spring bean又用到了J2EE Resource注解，Servlet中调用MerQueryInfoBusiness的方式是这样的：


~~~~
    Object obj = SpringUtil.getBean("merQueryInfoService");
~~~~

问题就可以找到了，SpringUitl.getBean时，用Spring bean注入一个CacheManager(因为CacheManager的scope为prototype，多例)，而@Resource注解又会注入一个Cachemanager，最后就看到了Spring初始化日志中CacheManager的两次初始化。
