---
layout: post
title: PropertySourcesPlaceholderConfigurer源码分析
category: program
---

在程序编码时，为了增加程序灵活度，配置数据一般都不会硬编码到代码中，而是在程序启动时，从配置中心获取，配置中心是一个统一管理系统配置的服务，项目不大时，配置中心可以是一个简单的配置文件，随着项目逐步状态，配置中心可以单独成为一个系统。

这里通过分析Spring的源码，看看最简单的文件配置，在Spring中是怎么实习的。

#### 类图

查看源码先从类图开始：

![image](http://afredlyj.github.io/assets/images/placeholder.png)

`PropertySourcesPlaceholderConfigurer`实现了`EnvironmentAware`，是`PlaceholderConfigurerSupport`的子类，而该父类实现了`BeanFactoryPostProcessor`接口，该接口关系到`PropertySourcesPlaceholderConfigurer`的主要流程。

#### BeanFactoryPostProcessor

`BeanFactoryPostProcessor`是Spring Bean生命周期中的一个扩展点，该接口会在IOC容器初始化之后，Bean初始化之前被调用：  

``` java
// AbstractApplicationContext
private void invokeBeanFactoryPostProcessors(
            Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

        for (BeanFactoryPostProcessor postProcessor : postProcessors) {
            postProcessor.postProcessBeanFactory(beanFactory);
        }
    }
```

#### placeholder处理过程

由上文可知，`PropertySourcesPlaceholderConfigurer`的入口函数在`postProcessBeanFactory`：

``` java
@Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        if (this.propertySources == null) {
            this.propertySources = new MutablePropertySources();
            // 加载环境相关的配置
            if (this.environment != null) {
                this.propertySources.addLast(
                    new PropertySource<Environment>(ENVIRONMENT_PROPERTIES_PROPERTY_SOURCE_NAME, this.environment) {
                        @Override
                        public String getProperty(String key) {
                            return this.source.getProperty(key);
                        }
                    }
                );
            }
            try {
            ／／ 加载资源文件并解析
                PropertySource<?> localPropertySource =
                    new PropertiesPropertySource(LOCAL_PROPERTIES_PROPERTY_SOURCE_NAME, this.mergeProperties());
                if (this.localOverride) {
                    this.propertySources.addFirst(localPropertySource);
                }
                else {
                    this.propertySources.addLast(localPropertySource);
                }
            }
            catch (IOException ex) {
                throw new BeanInitializationException("Could not load properties", ex);
            }
        }

        // 将配置信息设置到bean
        this.processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));
    }
```

重点关注配置信息解析之后，怎么设置到bean中，注意，这时候bean并没有初始化，所以只能修改BeanDefinition中的属性。关键代码如下：

``` java
    protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
            StringValueResolver valueResolver) {

        BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

        String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
        for (String curName : beanNames) {
            // Check that we're not parsing our own bean definition,
            // to avoid failing on unresolvable placeholders in properties file locations.
            if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
                BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
                try {
                    visitor.visitBeanDefinition(bd);
                }
                catch (Exception ex) {
                    throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage());
                }
            }
        }

        // New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
        beanFactoryToProcess.resolveAliases(valueResolver);

        // New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
        beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
    }

```

placeholder的处理，交给`BeanDefinitionVisitor`处理，需要注意的是参数`valueResolver`是一个实现了`StringValueResolver`的内部类：

``` java
// BeanDefinitionVisitor
public void visitBeanDefinition(BeanDefinition beanDefinition) {

        visitParentName(beanDefinition);
        visitBeanClassName(beanDefinition);
        visitFactoryBeanName(beanDefinition);
        visitFactoryMethodName(beanDefinition);
        visitScope(beanDefinition);
        visitPropertyValues(beanDefinition.getPropertyValues());
        ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
        visitIndexedArgumentValues(cas.getIndexedArgumentValues());
        visitGenericArgumentValues(cas.getGenericArgumentValues());
    }

// PropertySourcesPlaceholderConfigurer.processProperties
StringValueResolver valueResolver = new StringValueResolver() {
            public String resolveStringValue(String strVal) {
            // propertyResolver是一个PropertySourcesPropertyResolver对象
                String resolved = ignoreUnresolvablePlaceholders ?
                        propertyResolver.resolvePlaceholders(strVal) :
                    propertyResolver.resolveRequiredPlaceholders(strVal);
                return (resolved.equals(nullValue) ? null : resolved);
            }
        };

```

`BeanDefinitionVisitor`会逐个处理parentName, className, factoryBeanName, scope, propertyValues等数据，经过层层分析，placeholder最后在`PropertyPlaceholderHelper`中处理，具体的替代过程这里就不再分析，流程梳理清楚就好^_^。

总之，`PropertySourcesPlaceholderConfigurer`利用`BeanFactoryPostProcessor`和`BeanDefinition`实现参数的可配置化，这一切都发生的bean初始化之前。

#### context:property-placeholder标签

Spring默认提供了`context:property-placeholder`来支持`PropertySourcesPlaceholderConfigurer`，所以可以从Spring 相关的Schema入手：

``` java
//ContextNamespaceHandler
public void init() {
        registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
        registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
        registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
        registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
        registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
        registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
        registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
        registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
    }
```

该BeanDefinition的解析由`PropertyPlaceholderBeanDefinitionParser`完成。
