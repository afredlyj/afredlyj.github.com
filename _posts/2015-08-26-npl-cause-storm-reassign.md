---
layout: post  
title: 空指针导致storm集群重启
category: bug  
---

线上运行颇稳定的Storm 伪集群经过我折腾上线之后，会无缘故重启，这让我压力很大，因为这个版本的一个目标就是优化处理速度，降低消息丢失的概率，没想到上线之后整个服务都不稳定了，简直不能忍。

### 新版的修改点

#### 老版本的问题
CRM在线上运行了一年多，整个服务趋于稳定，但是偶尔还是会出现这样那样的问题：  
1. 老版本服务虽然稳定，但是偶尔会由于spouts取消息不及时，跟不上生产端数据入队列的速度，导致内存队列被撑满，最终导致搜集模块丢失很多数据；  
2. 支付行为监控有邮件报警业务，整个流程并没有异步解藕，导致支付监控相关的bolts处理很慢，长达200ms

#### 新版本解决方案

由于增加新的业务分析，所以决定在新增业务功能的同时，修复老版本存在的问题。  
1. 提供sputs／bolts的并行度，老版本默认是1；  
2. 邮件报警的逻辑涉及到网络请求，所以采用线程池，保证bolts的execute能尽快执行完毕；  
3. 不依赖storm的可靠性，storm可靠性的介绍可以参考[官网](https://storm.apache.org/documentation/Guaranteeing-message-processing.html)，需要注意的是，emit时取消消息id并没有关闭acker机制，需要将acker的数量手动设置为0，我是通过修改`storm.yaml`配置文件：
> topology.acker.executors: 0

改造之后，放到gamma压测，运行正常，消息处理正常，即使是高压环境，也只是在服务刚启动时会丢消息，于是窃喜。

#### 那么问题来了

压测正常，测试环境各种数据模拟也正常，是时候让它面对正式环境的风吹雨打了，于是我信心满满的安排上线，观察1小时，正常，2小时，正常，3小时，正常，嗯，下午就这么过去了，下班回家。
刚上地铁，“叮”——报警邮件响了，一看mq报警，消息堆积超过阈值，并且有增多的趋势，日了狗了。回到家赶紧看服务器，吓了一跳，worker一直在重启，由于新版主要调了并行度，所以首先就想到是不是这儿的原因，找到worker日志，由于业务的错误日志和storm的日志都在一个文件，所以看起来很麻烦，找了几个端口的日志，都是这样子的：

~~~~  
2015-08-24 15:13:04 backtype.storm.messaging.netty.Client [INFO] [New I/O boss #9] Reconnect ... [20]
2015-08-24 15:13:04 backtype.storm.messaging.netty.Client [WARN] [New I/O boss #9] Remote address is not reachable. We will close this client.
2015-08-24 15:13:04 backtype.storm.util [ERROR] [Thread-31] Async loop died!
java.lang.RuntimeException: java.lang.RuntimeException: Client is being closed, and does not take requests any more
        at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:107) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:78) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:77) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.disruptor$consume_loop_STAR_$fn__1577.invoke(disruptor.clj:89) ~[na:na]
        at backtype.storm.util$async_loop$fn__384.invoke(util.clj:433) ~[na:na]
        at clojure.lang.AFn.run(AFn.java:24) [clojure-1.4.0.jar:na]
        at java.lang.Thread.run(Thread.java:662) [na:1.6.0_41]
Caused by: java.lang.RuntimeException: Client is being closed, and does not take requests any more
        at backtype.storm.messaging.netty.Client.send(Client.java:125) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.daemon.worker$mk_transfer_tuples_handler$fn__4398$fn__4399.invoke(worker.clj:319) ~[na:na]
        at backtype.storm.daemon.worker$mk_transfer_tuples_handler$fn__4398.invoke(worker.clj:308) ~[na:na]
        at backtype.storm.disruptor$clojure_handler$reify__1560.onEvent(disruptor.clj:58) ~[na:na]
        at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:104) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        ... 6 common frames omitted
2015-08-24 15:13:04 backtype.storm.util [INFO] [Thread-31] Halting process: ("Async loop died!")  
~~~~  

看样子一致在试图重连其他bolts，由于日志有线程id，重连20次之后，worker挂掉，查看storm的重试策略配置：
>storm.messaging.netty.max_retries: 20

和上面的20次吻合，那么问题来了，是什么原因导致重连？是链接哪一方任务压力比较大，连接超时？还是说对端直接挂掉，导致连接断开？

事实证明，我当时分析的时候脑袋肯定被驴踢了，因为这个日志的上线文是: 
>java.io.IOException: Connection reset by peer  

所以很明显是对端的问题，继续跟踪supervisor的日志，有如下纪录：  

~~~~
2015-01-04 08:09:22 b.s.d.supervisor [INFO] [Thread-2] Shutting down and clearing state for id 3a19c10a-9c8a-44c7-ad1d-094295d000b5. Current supervisor time: 1420330161. State: :timed-out, Heartbeat: #bac
ktype.storm.daemon.common.WorkerHeartbeat{:time-secs 1420308003, :storm-id "CRMRealtimeServer-1-1419727321", :executors #{[5 5] [13 13] [21 21] [-1 -1]}, :port 6703}
2
~~~~

由supervisor的日志可知，supervisor和woker之间的连接超时，导致worker重启，所以猜想调整重试策略可能会有效，如果这个解释合理，那还有一个问题，为什么一个worker连接超时，会导致整个集群所有任务重新分配呢？

根据错误信息，我找到了storm的[jira](https://issues.apache.org/jira/browse/STORM-329)，大意是，两个worker，A和B，B订阅了A的消息，当B由于某种原因挂掉之后，A在获知最新的拓扑信息之前，一直往B发送消息，当到达最大重试次数时，worker A 抛出RuntimeException，从而退出。这个因为某个worker挂掉导致其他相关worker业挂掉的问题，可以通过升级storm版本解决。
但是得明确为什么第一个worker会挂，它挂掉的原因显然不是连接超时导致的，之前的分析，把因果关系弄反了，所以即使我调整了重试机制，也只是把服务重启的周期延长，并没有解决问题。

所以继续观察，由于业务和storm都是读取的storm/logback下的日志配置，为了查log更方便，关闭了业务日志输出，将storm的日志级别调整到info（别问我为什么这时候才想到...），接下来坐等服务重启。

黄天不负有心人（实在是自己太蠢），最后终于找到首先挂掉的worker，日志输出如下：  

~~~~

2015-08-20 20:06:28 backtype.storm.daemon.executor [ERROR] [Thread-23-securityChecker] 
java.lang.RuntimeException: java.lang.NullPointerException
        at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:107) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:78) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:77) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.daemon.executor$eval3918$fn__3919$fn__3931$fn__3978.invoke(executor.clj:745) ~[na:na]
        at backtype.storm.util$async_loop$fn__384.invoke(util.clj:433) ~[na:na]
        at clojure.lang.AFn.run(AFn.java:24) [clojure-1.4.0.jar:na]
        at java.lang.Thread.run(Thread.java:662) [na:1.6.0_41]
Caused by: java.lang.NullPointerException: null
        at com.keke.crm.analyzer.SecurityCheckRecord.analysis(SecurityCheckRecord.java:43) ~[stormjar.jar:na]
        at com.keke.crm.bolts.SecurityCheckBolt.execute(SecurityCheckBolt.java:38) ~[stormjar.jar:na]
        at backtype.storm.topology.BasicBoltExecutor.execute(BasicBoltExecutor.java:50) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        at backtype.storm.daemon.executor$eval3918$fn__3919$tuple_action_fn__3921.invoke(executor.clj:630) ~[na:na]
        at backtype.storm.daemon.executor$mk_task_receiver$fn__3839.invoke(executor.clj:398) ~[na:na]
        at backtype.storm.disruptor$clojure_handler$reify__1560.onEvent(disruptor.clj:58) ~[na:na]
        at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:104) ~[storm-core-0.9.1-incubating.jar:0.9.1-incubating]
        ... 6 common frames omitted
2015-08-20 20:06:28 backtype.storm.util [INFO] [Thread-23-securityChecker] Halting process: ("Worker died")
~~~~

日了狗了。是我一直没有理解storm的`fail-fast`，[这里](https://storm.apache.org/documentation/Fault-tolerance.html)讲了storm的错误容忍机制，也说明了worker心跳超时会导致任务重新分配。回到我的业务，由于`SecurityRecordBols`抛出`RuntimeException`，导致worker退出，又加上storm netty客户端超时处理的坑，导致整个storm挂掉。

所以最后的解决办法是代码判空，storm版本升级的事儿，先放着吧，为了防止以后出现`RuntimeException`导致整个storm挂掉，我在每个`nextTuple`和`execute`方法下面暴力的加上`try-catch-Exception`。
