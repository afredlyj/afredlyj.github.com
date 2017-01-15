---
layout: post
title: CountDownLatch源码分析
category: program
---

### AQS 简介

在介绍`CountDownLatch`之前，需要对`AQS`的基本结构作一些简单说明。该类是`java.util.concurrent`包的核心类之一，是并发包中很多同步类的基础，核心思想是通过一个共享变量来同步状态，子类根据自己的需求实现`AQS`的模版方法，通过这种方式维护共享变量的状态，模版方法如下：

* tryAcquire
* tryRelease
* tryAcquireShared
* tryReleaseShared
* isHeldExclusively

除了共享变量，AQS还需要维护一个阻塞队列，该队列是CHL队列的变种，CHL队列是一个非阻塞的FIFO队列，也就是说往里面插入或移除一个节点的时候，在并发条件下不会阻塞，而是通过自旋锁和 CAS 保证节点插入和移除的原子性队列。有三个核心属性：

```java
// 共享状态
private volatile int state;

// 头节点，head和tail表明这是一个双向队列
private transient volatile Node head;
// 尾节点
private transient volatile Node tail;
```

AQS就介绍到这里，再继续讲会比较枯燥，所以结合子类CountDownLatch倒推AQS的实现会相对比较好理解，AQS的权威介绍在[这里](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)，我看了一部分实在看不下去。

### await 方法

先看`await`方法，该方法可能会阻塞调用线程，如果计数器当前值为0，则调用线程立即返回，即不会阻塞，否则只有当满足以下条件时，该线程才会被唤醒：

1. 计数器为0；  
2. 等待线程中断或超时。

```java
   private final Sync sync;

   public void await() throws InterruptedException {
   		 // 获取共享锁，响应中断
        sync.acquireSharedInterruptibly(1);
    }
```

内部类`Sync`是`AbstractQueuedSynchronizer`的子类，实现了共享模式下锁的获取和释放，代码如下：

``` java

    /**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                // 当前线程调用getState返回1，不满足条件，继续往下之行
                if (c == 0)
                    return false;
                // 如果这时其他线程修改state变量为0，而此时c为1，nextc为0，statue变量为0，但是并不满足cap条件，当前线程执行下一次循环
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

```

我发现分析jdk的源码，一般都直接把源码和注释搬出来就好了，哈哈哈。分析`CountDownLatch`的代码也是如此，

```java
  public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
           // 先判断当前线程的中断状态
        if (Thread.interrupted())
            throw new InterruptedException();
            // 由子类CountDownLatch实现，返回值固定为1或者－1，判断state状态，如果state等于0，则返回1，否则返回－1
        if (tryAcquireShared(arg) < 0)
        
            doAcquireSharedInterruptibly(arg);
    }
    
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 将当前线程节点插入到FIFO的末尾
        // 流程1
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
        // 注意for循环的作用域，只有当获取成功或者线程被中断才会跳出循环
        // 流程2
            for (;;) {
            
                final Node p = node.predecessor();
                if (p == head) {
                // 尝试获取
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                    // 获取成功，则前继节点出队，同时传播到后继节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                // 流程3
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

分析到这里，基本上对`await`方法有了大致的了解。主要流程如下：
1. 将当前线程节点以共享模式加入到CLH队列中，进入流程2；  
2. 检查当前节点的前继节点，如果前继节点是头节点并且当前计数器为0，则将前继节点出队，并将当前节点设置为头节点，然后传递通知后继节点并退出返回，否则进入流程3；  
3. 检查当前线程是否应该阻塞（park），如果是就阻塞当前线程，直到其他线程调用unpark，被唤醒之后继续流程2；  
4. 只有当获取锁成功或者被中断时当前线程才回返回。 

上面的流程，需要注意的是，后继节点（也就起其他调用await方法的线程）被唤醒之后，也是走流程2的循环，如果满足条件就退出。

### countDown 方法

回到`countDown`方法，调用`Sync#releaseShared`，共享模式释放锁：

```java
   public void countDown() {
   		 // 内部类Sync实现AQS
        sync.releaseShared(1);
    }
```

```java
    public final boolean releaseShared(int arg) {
        tryReleaseShared由子类实现，CountDownLatch的实现见上文代码，通过CAS设置属性state的值，如果state为0，则返回true，否则返回false
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

`CountDownLatch`通过`FIFO`队列和`state`属性维护内部状态，初始化时由调用方设置`state`值，且必须大于0。

当线程调用`countDownd`时，state原子性减一，如果当前state等于0，则调用`doReleaseShared`方法，该方法是	AQS的私有方法，索性偷懒将注释和代码全部拷过来：

```java
    /**
     * Release action for shared mode -- signal successor and ensure
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
     */
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }

```

分析完`await`之后，其实`countDown`方法就比较简单，该方法就是在计数器为0时，唤醒头节点，头节点唤醒之后，根据`await`的分析，会传递到后继节点，这样会把整个阻塞队列唤醒。

#### 参考文档
1. http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-aqs-AbstractQueuedSynchronizer.html
2. http://www.idouba.net/sync-implementation-by-aqs/
3. http://www.blogjava.net/xylz/archive/2010/07/06/325390.html
4. http://gee.cs.oswego.edu/dl/papers/aqs.pdf 

