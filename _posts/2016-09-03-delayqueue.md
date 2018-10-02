---
layout: post
title: DelayQueue 源码分析
category: program
---

DelayQueue是一个无界的阻塞队列，从这个队列中取出来的元素都是过期的，head头是过期时间最长的元素。
DelayQueue =  BlockingQueue + PriorityQueue + Delayed。使用优先级队列实现阻塞队列，优先级的比较基准是时间。
 
#### 方法简介
1. add  
往队列中增加一个元素，底层调用`offer`方法
 
2. offer  
增加元素，有超时版本

3. put  
增加元素，由于DelayQueue是无界队列，所以该方法不会阻塞

4. poll  
从队列中取出head节点并从队列中删除，如果没有元素过期，则返回null，有超时版本

5. take  
从队列中取出head节点并从队列中删除，如果没有元素过期，则阻塞等待

6. peek  
从队列中取出head节点但并不删除，如果没有元素过期，则返回下一个即将过期的元素，实现时直接调用优先级队列的`peek`方法

#### 源码分析
`DelayQueue`成员变量如下：

```java  
    private transient final ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    // 没有使用volatile修饰，因为都在锁的范围内
    private Thread leader = null;
	 private final Condition available = lock.newCondition();

```  
`DelayQueue`个人理解是优先级队列的变种，元素按照超时时间排序，从头到尾，离超时时间越近的元素，越靠前排列，所以对队列中的元素有要求，`DelayQueue`队列的元素必须实现接口`Delayed`:

```java
public interface Delayed extends Comparable<Delayed> {

    /**
     * Returns the remaining delay associated with this object, in the
     * given time unit.
     *
     * @param unit the time unit
     * @return the remaining delay; zero or negative values indicate
     * that the delay has already elapsed
     */
    long getDelay(TimeUnit unit);
}
```

该接口继承了`Comparable`，作用是用于成员变量`PriorityQueue`的排序。
由于是无界队列，所以各个方法的超时版本和不带超时的版本是一样的，所以这里只分析不带超时版本的方法。

1. Leader/Follower模式


该模式简单来说，所有的工作线程分为Leader和Follower两种角色，只有一个Leader线程会处理任务，其他线程都在排队等待，称为Follower，当Leader获取到任务之后，通知其他Follower晋升为Leader，完成任务后等待下一次晋升为Leader，这样做的目的是减少线程切换对性能的影响。

2. offer方法  
`DelayQueue`使用`ReentrantLock`保证线程安全，增加元素时，直接往优先级队列中增加即可，过期判断都是在取元素时判断：

```java
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
            // 当有更早的过期元素成为队首元素时，通过设置leader为null，
            // 使当前leader线程失去leader位置，等待中的某个Follower线程成为leader
                leader = null;
            // 唤醒一个follower
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```

3. take方法  
该方法会阻塞当前线程，直到head元素不为空，对元素过期时间的判断，则在take方法中实现，除去LF模式的相关代码，take的伪代码如下：

```java
lock.lockInterruptibly();
try {
	for (;;) {
		E first = q.peek();
		if (first == null) {
		
		} else {
			delay = first.getDelay(TimeUnit.NANOSECONDS);
			if (delay <= 0) {
				return q.poll();
			}
		}
	}
} finally {
	lock.unlock
}
```

要注意的是，从优先级队列中取数据，使用`peek`，取出来之后需要对delay时间判断，如果未过期，则继续循环，否则调用`poll`取出队首元素。另外，调用`Delayed`时，使用`TimeUnit.NANOSECONDS`，在实现自己的`Delayed`时需要注意这个细节。

接下来看看`take`方法中的LF模式：

```java

public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    // 判断是否到期
                    if (delay <= 0)
                        return q.poll();
                        
                    // 队首元素未到期
                    // 释放引用
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                    // 已经设置leader，follower线程无限期wait，并释放锁
                    // 注意这里的await方法没有超时时间，说明已经有leader在工作
                        available.await();
                    else {
                    // 未设置leader，当前线程成为leader
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                        // leader等待队首元素到期，注意await方法有超时时间，并释放锁
                            available.awaitNanos(delay);
                        } finally {
                        // await超时结束之后，如果当前线程还是leader，则更换leader，
                        // 当前线程进入下一次for循环，并尝试取出队首元素
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
            // take返回前，如果leader为null且队列不为空，则发送available信号
                available.signal();
            lock.unlock();
        }
    }
```

#### 参考资料
1. http://www.10tiao.com/html/308/201511/400583280/1.html
2. http://stackoverflow.com/questions/3058272/explain-leader-follower-pattern  
3. http://stackoverflow.com/questions/8119727/leader-follower-vs-work-queue  
4. http://www.kircher-schwanninger.de/michael/publications/lf.pdf
5. http://www.dczou.com/viemall/319.html



