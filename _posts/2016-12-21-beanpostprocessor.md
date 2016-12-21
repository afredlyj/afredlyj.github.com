---
layout: post
title: BeanPostProcessor 分析
category: program
---

前面分析了Spring的PropertySourcesPlaceholderConfigurer，这篇文章分析`BeanPostProcessor`，由于是接口，本文会使用Spring框架本身的实现类做一下简单分析，包括`BeanNameAutoProxyCreator`和`AutowiredAnnotationBeanPostProcessor`。

`BeanPostProcessor`是Spring支持的扩展点，通过该接口可以动态修改bean，比如生成代理对象，执行时机如下图：
![image](http://afredlyj.github.io/assets/images/spring-bean2.png)

该接口定义如下：

```java 
// 在对象初始化之前调用。
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

// 在对象初始化之后调用。
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

```

#### BeanNameAutoProxyCreator

`BeanNameAutoProxyCreator`类就是`BeanPostProcessor`实现类之一，该类会为匹配的对象自动创建代理对象，并将该代理对象返回，之后应用中使用的都是该代理对象。该类的类图如下：

![image](http://afredlyj.github.io/assets/images/beannameautoproxycreator.png)

在对`BeanNameAutoProxyCreator`分析之前，需要回顾Spring Bean的生命周期，其中牵涉到`BeanPostProcessor`的扩展时机。

```java
// AbstractAutowireCapableBeanFactory.java

@Override
	protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		// Make sure bean class is actually resolved at this point.
		resolveBeanClass(mbd, beanName);

		// Prepare method overrides.
		try {
			mbd.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			// 扩展点
			Object bean = resolveBeforeInstantiation(beanName, mbd);
			// 如果该方法执行完成之后bean非空，则执行返回，不会继续执行下面的扩展点逻辑
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		Object beanInstance = doCreateBean(beanName, mbd, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (mbd.hasBeanClass() && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				bean = 
				// 执行InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation，bean实例化之前的逻辑
				applyBeanPostProcessorsBeforeInstantiation(mbd.getBeanClass(), beanName);
				if (bean != null) {
				// 如果返回不为空，则继续执行BeanPostProcessor.postProcessAfterInitialization
					bean = 
		applyBeanPostProcessorsAfterInitialization(bean, beanName);
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}

protected Object applyBeanPostProcessorsBeforeInstantiation(Class beanClass, String beanName)
			throws BeansException {

// 遍历当前可用的BeanPostProcessor
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
				if (result != null) {
					return result;
				}
			}
		}
		return null;
	}	

```

使用默认的`BeanNameAutoProxyCreator`时，上述方法返回null，所以有了下文的扩展点，我们这里直接跳过部分扩展点，直接执行到关键代码处。

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				public Object run() {
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
		// 如果当前的bean实现了Aware，则调用相关方法
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
		// 扩展点，bean初始化之前调用
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
		// 执行初始化方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}

		if (mbd == null || !mbd.isSynthetic()) {
		// 扩展点
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
	}
```

`BeanNameAutoProxyCreator`通过上文中的最后一个扩展点将目标bean封装成代理对象，具体代码如下：

```java
// AbstractAutoProxyCreator.java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.containsKey(cacheKey)) {
			// 判断是否需要封装，如果需要则会创建代理对象
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

执行完`wrapIfNecessary`之后，`BeanNameAutoProxyCreator`就返回目标对象的代理对象，代理对象的生成逻辑这里就不再分析，生成的代理对象beanName和目标对象一致，之后业务代码调用的就是生成的代理对象。

参考文档：http://jinnianshilongnian.iteye.com/blog/1492424

// TODO

参考文档：http://www.cnphp6.com/archives/85639