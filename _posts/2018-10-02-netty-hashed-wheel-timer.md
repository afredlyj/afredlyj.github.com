---
layout: post
title: Netty 时间轮HashedWheelTimer 源码分析
category: netty
---

#### 业务场景
平时开发中，有这样的场景：
> 在用户下单成功30分钟后，给用户发送确认短信。

实现这个功能有如下几种方案：  
1. 定时轮询数据库，取出满足条件的数据然后逐个处理，注意轮询的时间间隔；  
2. 采用JDK提供的延迟队列，消费者在指定时间之后才能取到过期消息，现成的类：[DelayQueue](https://afredlyj.github.io/posts/delayqueue.html)；  
3. 采用时间轮，Netty实现了自己的时间轮算法：`HashedWheelTimer`；
4. 利用消息中间件实现，比如利用RabbitMQ的消息过期时间和死信队列可以实现相似功能；


#### 时间轮原理

部分摘抄网络文章：

>George Varghese 和 Tony Lauck 1996 年的论文：[Hashed and Hierarchical Timing Wheels: data structures to efficiently implement a timer facility](https://cseweb.ucsd.edu/~varghese/PAPERS/twheel.ps.Z)提出了一种定时轮的方式来管理和维护大量的Timer调度算法.Linux 内核中的定时器采用的就是这个方案。

见名知意，`HashedWheelTimer`逻辑上是一个环形结构，可以想象成时钟，分为很多槽位，一个槽位代表一个单位时间（单位时间越短，时间轮的精度越高），槽位上是一个保存到期任务的集合，同时一个指针随着时间流逝一格一格转动，并执行对应集合中所有到期的任务。任务通过取模决定应该放入哪个槽位。

> 环形结构可以根据超时时间的 hash 值(这个 hash 值实际上就是ticks & mask)将 task 分布到不同的槽位中, 当 tick 到那个槽位时, 只需要遍历那个槽位的 task 即可知道哪些任务会超时(而使用线性结构, 你每次 tick 都需要遍历所有 task), 所以, 我们任务量大的时候, 相应的增加 wheel 的 ticksPerWheel 值, 可以减少 tick 时遍历任务的个数.

#### 构造函数

##### 构造函数的作用

1. 构造时间轮，槽位数会向上对齐，2的n次方，环形结构用数组实现，每个槽位的任务集合保存在并发Set中
2. 初始化相关配置
3. 使用线程池创建一个线程，但并不启用


##### 代码分析

`HashedWheelTimer`提供多个重载的构造函数，最终会调用：

```java
public HashedWheelTimer(
            ThreadFactory threadFactory,
            long tickDuration, TimeUnit unit, int ticksPerWheel)
```

其中入参说明如下：

>threadFactory : 线程池，虽然引入了线程池，但是最终只会创建一个线程；  
>tickDuration : 每个tick的时间，即单位时间；  
>unit : 指定`tickDuration`的时间单位；  
>ticksPerWheel : 每个时间轮的tick总数，该值越大，占用内存可能会越多

构造函数的核心代码如下：

```java
// constructor start 
  // Normalize ticksPerWheel to power of two and initialize the wheel.
  // 对齐，生成的格子数量为2的n次方
  wheel = createWheel(ticksPerWheel);
  // 由于wheel.length等于2的n次方，mask的二进制低位全是1，常规操作
  mask = wheel.length - 1;

  // Convert tickDuration to nanos.
  this.tickDuration = unit.toNanos(tickDuration);

  // Prevent overflow.
  if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
      throw new IllegalArgumentException(String.format(
              "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
              tickDuration, Long.MAX_VALUE / wheel.length));
  }

  // 创建Worker线程，此时并没有启动，如果此时没有任务，线程空转会有一定的资源浪费
  workerThread = threadFactory.newThread(worker);
// constructor end


private static Set<HashedWheelTimeout>[] createWheel(int ticksPerWheel) {
    if (ticksPerWheel <= 0) {
        throw new IllegalArgumentException(
                "ticksPerWheel must be greater than 0: " + ticksPerWheel);
    }
    if (ticksPerWheel > 1073741824) {
        throw new IllegalArgumentException(
                "ticksPerWheel may not be greater than 2^30: " + ticksPerWheel);
    }

    ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
    Set<HashedWheelTimeout>[] wheel = new Set[ticksPerWheel];
    for (int i = 0; i < wheel.length; i ++) {
        wheel[i] = Collections.newSetFromMap(
                PlatformDependent.<HashedWheelTimeout, Boolean>newConcurrentHashMap());
    }
    return wheel;
}

// 对齐格子数量，稍大于ticksPerWheel且是2的n次方
private static int normalizeTicksPerWheel(int ticksPerWheel) {
    int normalizedTicksPerWheel = 1;
    while (normalizedTicksPerWheel < ticksPerWheel) {
        normalizedTicksPerWheel <<= 1;
    }
    return normalizedTicksPerWheel;
}
```

#### 添加任务

添加任务的时间复杂度为O(1)，提供的方法如下：

##### 方法签名

通过如下方法向`HashedWheelTimer`增加一个延迟任务，该任务只会执行一次：

```java
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit);
```

入参：

```java
task : 任务的实现
delay : 指定延迟时间
unit : 延迟时间的时间单位
```

##### 执行流程

1. 首先判断worker线程的状态，如果线程未启动，则启动线程，之后阻塞等待`HashedWheelTimer`初始化完成，使用`AtomicInteger`保证线程只会被执行一次；
2. 获取读锁
3. 初始化一个`HashedWheelTimeout`，并计算槽位放到指定槽位对应的任务集合中
4. 释放读锁
5. 返回上面创建的`HashedWheelTimeout`对象，客户端可以利用这个timeout对象取消任务

##### 代码分析

简述完执行流程，接下来结合代码分析，核心流程如下：

```java
// newTimeout 函数
// 理论上的任务执行时间和时间轮启动时间之间的时间差
long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
  // Add the timeout to the wheel.
  HashedWheelTimeout timeout;
  lock.readLock().lock();
  try {
  // 在读锁之后调用构造函数
      timeout = new HashedWheelTimeout(task, deadline);
      if (workerState.get() == WORKER_STATE_SHUTDOWN) {
          throw new IllegalStateException("Cannot enqueue after shutdown");
      }
      wheel[timeout.stopIndex].add(timeout);
  } finally {
      lock.readLock().unlock();
  }
  
 // HashedWheelTimeout 类
 // 任务状态标志
private static final int ST_INIT = 0;
private static final int ST_CANCELLED = 1;
private static final int ST_EXPIRED = 2;


private final TimerTask task;	// 具体任务
final long deadline;	// 同上面的deadline
final int stopIndex;	// 槽位
volatile long remainingRounds;	// 剩余轮次，有可能当前任务需要时间轮回多次，如果大于0，则本次不执行，需要使用volatile保证可见性
private final AtomicInteger state = new AtomicInteger(ST_INIT);

HashedWheelTimeout(TimerTask task, long deadline) {
    this.task = task;
    this.deadline = deadline;
	// 计算槽位
   long calculated = deadline / tickDuration;
   // 已经过期的任务，存入当前槽位直接，方便worker在方法调用完后执行（添加和执行分别使用读写锁，可能当前线程在读写锁竞争失败，槽位calculated的任务已经执行完成，此时将任务添加到当前槽位tick中）
    final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
    stopIndex = (int) (ticks & mask);
    // 可能小于等于0，如果是小于等于0，Worker会在当前tick中执行
    remainingRounds = (calculated - tick) / wheel.length;
}
```

#### 执行任务

##### 主要流程
1. 设置`HashedWheelTimer`初始化标志；
2. 计算下一个槽的执行时间，并等待，sleep结束后返回，返回值为Timer启动后到这次tick所过去的时间
3. 获取写锁
4. 轮询当前槽位中的所有任务，如果remainingRounds<=0，则表示任务到期需要执行，此时由于在写锁范围内，所以只是将任务添加到`expiredTimeouts`队列中，否则任务的remainingRounds自减
5. 轮询完成之后，tick加一
6. 释放写锁

##### 代码分析
Worker实现了`Runnable`接口：

```java
// run 方法
do {
    final long deadline = waitForNextTick();
    if (deadline > 0) {
        fetchExpiredTimeouts(expiredTimeouts, deadline);
        notifyExpiredTimeouts(expiredTimeouts);
    }
} while (workerState.get() == WORKER_STATE_STARTED);

// waitForNextTick 方法
/**
    * calculate goal nanoTime from startTime and current tick number,
    * then wait until that goal has been reached.
    * @return Long.MIN_VALUE if received a shutdown request,
 * current time otherwise (with Long.MIN_VALUE changed by +1)
 */
 // sleep，直到下一个tick到来
private long waitForNextTick() {
    long deadline = tickDuration * (tick + 1);

    for (;;) {
        final long currentTime = System.nanoTime() - startTime;
        //计算需要sleep的时间, 之所以加9999999后再除10000000, 是因为保证为10毫秒的倍数.
        long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

        if (sleepTimeMs <= 0) {
            if (currentTime == Long.MIN_VALUE) {
                        return -Long.MAX_VALUE;
            } else {
                return currentTime;
            }
        }

        // Check if we run on windows, as if thats the case we will need
        // to round the sleepTime as workaround for a bug that only affect
                // the JVM if it runs on windows.
        //
        // See https://github.com/netty/netty/issues/356
        if (PlatformDependent.isWindows()) {
            sleepTimeMs = sleepTimeMs / 10 * 10;
        }

        try {
            Thread.sleep(sleepTimeMs);
                } catch (InterruptedException e) {
            //当调用Timer.stop时, 退出
            if (workerState.get() == WORKER_STATE_SHUTDOWN) {
                return Long.MIN_VALUE;
            }
        }
    }

private void fetchExpiredTimeouts(
            List<HashedWheelTimeout> expiredTimeouts, long deadline) {

    // Find the expired timeouts and decrease the round counter
        // if necessary.  Note that we don't send the notification
        // immediately to make sure the listeners are called without
        // an exclusive lock.
        lock.writeLock().lock();
        try {
            fetchExpiredTimeouts(expiredTimeouts, wheel[(int) (tick & mask)].iterator(), deadline);
        } finally {
            // Note that the tick is updated only while the writer lock is held,
            // so that newTimeout() and consequently new HashedWheelTimeout() never see an old value
            // while the reader lock is held.
            tick ++;
            lock.writeLock().unlock();
        }
    }

private void fetchExpiredTimeouts(
            List<HashedWheelTimeout> expiredTimeouts,
            Iterator<HashedWheelTimeout> i, long deadline) {

    while (i.hasNext()) {
            HashedWheelTimeout timeout = i.next();
            if (timeout.remainingRounds <= 0) {
                i.remove();
                if (timeout.deadline <= deadline) {
                    expiredTimeouts.add(timeout);
                } else {
                    // The timeout was placed into a wrong slot. This should never happen.
                    throw new Error(String.format(
                            "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                }
            } else {
                timeout.remainingRounds --;
            }
        }
    }

private void notifyExpiredTimeouts(
            List<HashedWheelTimeout> expiredTimeouts) {
        // Notify the expired timeouts.
        for (int i = expiredTimeouts.size() - 1; i >= 0; i --) {
        // 同步轮训调用，根据下文贴出的HashedWheelTimeout#expire方法，如果TimerTask的run方法是耗时操作，则会影响Worker线程的执行，所以应该在TimerTask中封装一层，将业务逻辑扔到业务线程池中执行
            expiredTimeouts.get(i).expire();
        }

    // Clean up the temporary list.
        expiredTimeouts.clear();
    }
    
 // HashedWheelTimeout 类
 public void expire() {
    if (!state.compareAndSet(ST_INIT, ST_EXPIRED)) {
        return;
    }

    try {
        task.run(this);
    } catch (Throwable t) {
        if (logger.isWarnEnabled()) {
            logger.warn("An exception was thrown by " + TimerTask.class.getSimpleName() + '.', t);
        }
    }
}
```

以上即为时间轮的执行流程，时间复杂度最差情况下为O(n)，即所有的任务都在同一个槽位中。

#### 取消任务

任务添加之后返回一个`Timeout`对象，该对象提供`cancel`方法：

```java
@Override
public boolean cancel() {
    if (!state.compareAndSet(ST_INIT, ST_CANCELLED)) {
        return false;
    }

    wheel[stopIndex].remove(this);
    return true;
}

```

直接将任务从槽位的任务集合中删除，时间复杂度也是O(1)。

#### 参考资料
1. https://www.jianshu.com/p/7beebbc61229
2. https://my.oschina.net/haogrgr/blog/490348
3. https://my.oschina.net/haogrgr/blog/490266
4. https://my.oschina.net/haogrgr/blog/489320
5. https://zacard.net/2016/12/02/netty-hashedwheeltimer/

