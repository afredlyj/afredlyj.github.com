---
layout: post  
title: ActiveMQ 使用搜集  
category: program  
---  


第一次使用ActiveMQ（以下简称mq）是在支付通知服务中，当时草草封装之后，没有压力测试，就直接用到正式服务中，由于支付订单量并不大，所以线上一直没有问题，后来将这个工具包用到 CRM 系统中，就发现问题频繁出现，吃了一回哑巴亏。

现在将之前使用过程中忽略的地方记录下来，方便以后查阅。在整个过程中，并没有翻阅ActiveMQ的源码，仅仅依靠官方文档和自己编写的示例代码总结得来，可能有些地方还是理解不到位，还需要继续深究。

##Producer Flow Control##

mq自己实现了Flow Control（流量控制，默认开启），在mq的版本中，4.x和5.x流量控制实现原理并不相同，前者通过 TCP Flow Control 实现流量控制，只能针对链接，而5.x之后的PFC，能针对某个特定的producer，这里只讨论5.x。在5.0中，broker通过检测内存或者文件大小，判断是否已经达到容量上限，如果到达上限，broker就会减慢对消息的处理。默认情况下，producer会阻塞，不会再将消息发送到broker，直到broker有空闲容量。如果发现管理后台消息消费比较慢，甚至有消息堆积，可以尝试从这方面入手。

需要注意的是，默认情况下，broker流量控制的整个过程producer端并不会有异常日志，加大了对这种异常情况排查的难度。可以在broker的配置文件中设置`sendFailNoSpaceAfterTimeout`或`sendFailNoSpace`，设置之后，如果broker容量不足，producer端就能捕获到`javax.jms.ResourceAllocationException`，配置方法如下：

~~~~  
<systemUsage>
 <systemUsage sendFailIfNoSpace="true">
   <memoryUsage>
     <memoryUsage limit="20 mb"/>
   </memoryUsage>
 </systemUsage>
</systemUsage>

<systemUsage>
 <systemUsage sendFailIfNoSpaceAfterTimeout="3000">
   <memoryUsage>
     <memoryUsage limit="20 mb"/>
   </memoryUsage>
 </systemUsage>
</systemUsage>  
~~~~  

当producer端采用异步发送时，并不会等待broker的确认消息，所以默认情况下，即使达到容量上限，producer仍然没法知晓，可以通过改变connection的`producerWindowSize`属性修改默认配置：

~~~~  

// 通过代码配置
ActiveMQConnectionFactory connctionFactory = ...
connctionFactory.setProducerWindowSize(1024000);

// 通过url配置
tcp://127.0.0.1:61616?jms.useAsyncSend=true&jms.producerWindowSize=1024000

~~~~  

producerWindowSize官方解释如下：

>The ProducerWindowSize is the maximum number of bytes of data that a producer will transmit to a broker before waiting for acknowledgment messages from the broker that it has accepted the previously sent messages.

即，在producer发送`producerWindowSize`字节的数据后，broker会回包通知producer，在此之前的消息都已经被broker接收。

如果使用同步发送，或者异步发送但配置了producerWindowSize属性，一旦达到容量上限，broker会阻塞当前producer，而不是整个链接，当有空闲容量时，broker会回一个`ProducerAck`。如果producer继续发送消息，broker将阻塞整个connection，倘若此时的consumer和producer共用同一个connection，将会导致死锁。

##异步和同步发送##

mq提供消息同步和异步发送两种方式，如果能接受少量消息丢失，可以采用异步发送，否则用同步。[这里](http://activemq.apache.org/how-do-i-enable-asynchronous-sending.html)大致介绍了mq同步发送消息的处理流程。

##Prefetch limit##

####机制介绍####

mq为了提高吞吐量，采取prefetch 机制，即consumer端会在内存中维护一个消息的缓冲区，存放需要消费的消息，通过这种方式，consumer并不需要每条消息都主动请求（poll）broker，而是broker会push定量的消息到consumer端，以此降低频繁网络请求导致的性能损耗。

这种机制存在风险，尤其是consumer的消息处理速度跟不上broker push的速度时，这会导致大量消息充斥consumer的缓冲区，可能出现一个consumer繁忙，另一个consumer空闲的情况，并不利于消息及时处理。因此，mq提供`prefetch limit`参数限制每次push到consumer的消息数量。当达到`prefetch limit`上限，consumer不会再收到消息，直到consumer给broker发送确认消息。

如果consumer消息处理足够快，可以调大limit上限，这样整个系统的性能会比较可观。当该参数设置为0时，表示关闭prefect，consumer每次主动从broker拉取（poll）消息，而不是broker将消息push到consumer。mq不同服务的默认prefetch limit值如下：

>persistent queues (default value: 1000)  
>non-persistent queues (default value: 1000)  
>persistent topics (default value: 100)  
>non-persistent topics (default value: Short.MAX_VALUE -1)

我一般都是通过broker url指定limit大小：

~~~~  
tcp://localhost:61616?jms.prefetchPolicy.queuePrefetch=100
~~~~  

将每个队列prefetch limit设置为100，即broker每次都会将100个消息push到consumer端。

当使用`DefaultMessageListenerContainer`时，prefetch可能会引起问题，官方文档是这么说的：

>Consuming messages from a pool of consumers an be problematic due to prefetch. Unconsumed prefetched messages are only released when a consumer is closed, but with a pooled consumer the close is deferred (for reuse) till the consumer pool closes. This leaves prefetched messages unconsumed till the consumer is reused. This feature can be desirable from a performance perspective but it can lead to out-of-sequence messages when there is more than one consumer in the pool.
For this reason, the org.apache.activemq.pool.PooledConnectionFactory does not pool consumers. 
The problem is visible with the Spring DMLC when the cache level is set to CACHE_CONSUMER and there are multiple concurrent consumers.
One solution to this problem is to use a prefetch of 0 for a pooled consumer, in this way, it will poll for messages on each call to receive(timeout). Another option is to enable the AbortSlowAckConsumerStrategy on the broker to disconnect consumers that have not acknowledged a Message after some configurable time period.

####测试demo####

为了测试prefetch对consumer的影响，写了一些测试代码，具体项目代码在[这里](https://github.com/afredlyj/activeMQDemo)。

~~~~

// applicationContext-without-dmlc.xml
    <bean id="normalSendConnectionFactory" class="org.apache.activemq.pool.PooledConnectionFactory"
          destroy-method="stop">
        <property name="connectionFactory">
            <bean class="org.apache.activemq.ActiveMQConnectionFactory">
                <property name="brokerURL">
                    <value>${brokerUrl}</value>
                </property>
            </bean>
        </property>
    </bean>

    <bean id="normalJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="normalSendConnectionFactory"></property>
        <property name="receiveTimeout" value="600"></property>
        <property name="sessionAcknowledgeMode">
            <value>2</value>
        </property>
        <property name="deliveryPersistent">
            <value>true</value>
        </property>
    </bean>
    
~~~~

由于需要提前添加大量消息到broker，所以`applicationContext-without-dmlc.xml` 文件中并没有设置DefaultMessageListenerContainer节点。
 
~~~~  
// applicationContext-with-sleep-receiver.xml

	<bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
		<property name="brokerURL">
			<value>${brokerUrl}?jms.prefetchPolicy.queuePrefetch=100</value>
		</property>
	</bean>


    <bean id="example.MyQueue" class="org.apache.activemq.command.ActiveMQQueue">
		<constructor-arg index="0" value="example.MyQueue" />
	</bean>

    <!-- dead letter queue -->
    <bean id="ActiveMQ.DLQ" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg index="0" value="ActiveMQ.DLQ" />
    </bean>
	
	<!-- this is the Message Driven POJO (MDP), singleton -->
	<bean id="messageListener" class=" afred.jms.activeMQDemo.example.prefetch.SleepMQMsgReceiver" />

	<bean id="myExceptionListener" class="afred.jms.activeMQDemo.receive.exceptionhandler.MQExceptionListener"></bean>

	<bean id="jmsContainer"
		class="org.springframework.jms.listener.DefaultMessageListenerContainer">
		<property name="connectionFactory" ref="connectionFactory" />
		<property name="destination" ref="example.MyQueue" />
		<property name="messageListener" ref="messageListener" />
		<property name="maxConcurrentConsumers" value="2"></property>
		<property name="exceptionListener" ref="myExceptionListener"></property>
        <!--<property name="sessionTransacted" value="true" />-->
        <!--<property name="transactionManager" ref="local.transactionManager" />-->
	</bean>
	
~~~~

`applicationContext-with-sleep-receiver.xml`文件是consumer端的配置文件，messageListener是单例，并将最大consumer并发数设置2，默认是1，由于consumer由DMLC管理，测试过程中只能通过日志框架打印线程ID，以此观察两个consumer的消息处理过程。在`brokerURL`中添加了`jms.prefetchPolicy.queuePrefetch=100`属性，测试代码如下：

~~~~

    @Before
    public void addMessages() {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("applicationContext-without-dmlc.xml");
        JmsTemplate jmsTemplate = (JmsTemplate) context.getBean("normalJmsTemplate");
        int times = 150;
        ISendTest normalTest = new NormalMQSend("example.MyQueue", times, jmsTemplate);
        normalTest.run();
        System.out.println("add message to activemq finished.");
    }

    @org.junit.Test
    public void fetchMessage() {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("applicationContext-with-sleep-receiver.xml");
        context.start();

        try {
            TimeUnit.MINUTES.sleep(5);
        } catch (Exception e) {
            e.fillInStackTrace();
        }

//        assertEquals(true, true);
    }
~~~~

在consumer消费消息之前，向mq中添加150条消息，依据prefetch limit为100，猜想broker push到两个consumer的消息数量应该分别是100和50，运行测试用例后，根据logback打印的线程id消费记录，证实了猜想。

也就是说，broker会一次性将prefetch limit数量的消息push到一个consumer，这会导致当前consumer压力增大，同时其他consumer由于broker没有可用消息而空闲，这显然不符合常规。


##Spring + JmsTemplate 收发消息##

现在线上使用`JmsTemplate.send`发送消息到broker，并使用`DefaultMessageListenerContainer`接收消息，并没有发现异常，相比之前的版本，总结了使用过程中应该注意的点。

####使用JmsTemplate 需要注意的点####

mq官网关于JmsTemplate使用过程中应该注意的点有详细[说明](http://activemq.apache.org/jmstemplate-gotchas.html)。我这里自己再总结一次。

0. 每次调用JmsTemplate的发送和接收方法都会创建connection、session、producer或consumer   
这种特性在消息量并不大的时候并不会对服务产生较大影响，一旦消息量增大，由于需要频繁的创建网络链接，从而导致服务性能急剧下降，甚至影响到正常业务。之前一次支付改版，就是因为没有池化网络链接，在下班高峰期producer端几近崩溃，所以这里需要做两件事：  
> * producer端的connectionFactory由普通的`ActiveMQConnectionFactory`改为`PooledConnectionFactory`或`CachingConnectionFactory`，同时注意调整这两个connectionFactory的参数设置。  
> * 将consumer端改为`DefaultMessageListenerContainer`接收。 

0. JmsTemplate.send 默认采用同步发送  
即使按照上一条，使用`PooledConnectionFactory`或`CachingConnectionFactory`后，测试发现producer端的性能提升并不明显，问题在于，默认情况下，send方法会调用`ActiveMQConnection.syncSendPacket`，也就是走的同步发送。producer在发送消息到broker，broker将消息持久化，并返回`ProducerAck`后，此次send操作才算完成，如果不计较少量消息丢失，可以配置brokerURL，使用异步发送：  
~~~~  
   tcp://127.0.0.1:61616?jms.useAsyncSend=true&jms.producerWindowSize=1024000  
~~~~  
 
0. 尽量避免使用JmsTemplate.receive方法  
如果当前broker中没有消息，consumer端调用JmsTemplate.receive方法会阻塞，虽然可以设置receive的超时时间，但是根据调用一次，创建一个新connection、session和consumer的尿性，原生的recevie方法简直鸡肋。
另外，由于prefetch机制的作用，同时receive方法每次调用之后就会close掉当前的connection、session和consumer，会浪费网络带宽，比如，设置prefetch limit为1000，当broker中消息较多时（ > 1000），broker将会push 1000条消息到consumer端，由于receive每次处理一条消息，剩下的999条消息即使到达consumer端的缓冲区中，仍然无法消息，broker仍然需要重递这些消息（可以根据这些消息的`JMSXDeliveryCount`属性值判断该消息是否被重递），如此反复，显然会浪费很多资源。

####DefaultMessageListenerContainer的使用####

`DefaultMessageListenerContainer`支持动态扩容，另外，使用DMLC时，不要使用`PooledConnectionFactory`或`CachingConnectionFactory`，而应该将资源管理交给它自己处理。详细说明可以参考[官方文档](http://docs.spring.io/spring/docs/3.2.7.RELEASE/javadoc-api/org/springframework/jms/listener/DefaultMessageListenerContainer.html)。

如果并不想在Spring配置文件中初始化DMLC，而偏向于在代码中创建，那么在创建之后需要初始化，否则container并不能接收消息，正确的做法是调用container的`afterPropertiesSet`和`start`方法。

##总结##

这篇文章只记录了ActiveMQ的基本用法，以及会影响服务性能的几个点，但是，对消息的持久化、ActiveMQ集群并没有深入了解，这些都是以后需要研究的点。





