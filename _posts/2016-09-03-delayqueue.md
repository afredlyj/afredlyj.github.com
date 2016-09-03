---
layout: post
title: DelayQueue 源码分析
category: program
---

DelayQueue是一个无界的阻塞队列，从这个队列中取出来的元素都是过期的，head头是过期时间最长的元素。
 
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

参考文档：  
http://stackoverflow.com/questions/3058272/explain-leader-follower-pattern  
http://stackoverflow.com/questions/8119727/leader-follower-vs-work-queue  
http://www.kircher-schwanninger.de/michael/publications/lf.pdf

该模式简单来说，所有的工作线程分为Leader和Follower两种角色，只有一个Leader线程会处理任务，其他线程都在排队等待，称为Follower，当Leader获取到任务之后，通知其他Follower晋升为Leader，完成任务后等待下一次晋升为Leader。

2. offer方法  
`DelayQueue`使用`ReentrantLock`保证线程安全，增加元素时，直接往优先级队列中增加即可，过期判断都是在取元素时判断：

```java
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
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




