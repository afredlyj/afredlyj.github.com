---
layout: post
title: ReentrantLock源码分析
category: program
---

ReentrantLock提供的功能和`synchronized`类似，但是更具灵活性，是一种可重入的互斥锁。可重入表示ReentrantLock能够支持一个线程对资源的重复加锁，互斥表示一次只能有一个线程成功获取到锁，另外，ReentrantLock在获取锁时还提供公平和非公平两个选择。

ReentrantLock使用方式如下：

```java
 class X {
    private final ReentrantLock lock = new ReentrantLock();
    // ...
 
    public void m() {
      lock.lock();  // block until condition holds
      try {
        // ... method body
      } finally {
        lock.unlock()
      }
    }
 }
```
ReentrantLock实现了Lock接口，主要方法有`lock()`, `unlock()`, `lockInterruptibly`, `tryLock`等。

借助于AQS，ReentrantLock实现了互斥锁的逻辑，由于在[CountDownLatch源码分析](https://afredlyj.github.io/posts/countdownlatch.html)中已经分析了AQS的源码，本篇不再详述。

### 构造函数

ReentrantLock提供两种不同的锁：公平锁和非公平锁，公平锁是指获取锁时符合时间上的绝对顺序，即FIFO。

```java
 public ReentrantLock() {
        sync = new NonfairSync();
    }
    
 public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

默认选择非公平锁，`NonfairSync`和`FairSync`都是AQS的子类，锁的逻辑都代理给Sync实现。

### 公平锁

我按照公平与否的维度分析ReentrantLock的几个主要方法

#### lock

由上文的构造方法可知，sync在ReentrantLock创建时初始化。

```java
   public void lock() {
        sync.lock();
    }
    
   static final class FairSync extends Sync {
   		final void lock() {
            acquire(1);
        }
        
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
            	// 当前没有线程获取到锁
            	// 如果当前节点不存在前置节点，即在此刻之前没有线程在等待获取锁，则设置state，CAS成功之后，设置拍他锁所有者线程
            	
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
            // 有线程成功获取锁，并且就是当前的调用线程
            // 表明当前线程重入，累计state
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
   }
```

由于是互斥锁，通过判断state为0，就可知是否有线程成功获取锁，如果为0且没有前置节点，则通过CAS改变state的值，如果不为0，则判断获取锁的线程是否就是当前线程，如果是，则增加state计数，表示线程的重入次数增加，否则获取锁失败。

#### unlock

释放锁时，公平锁和非公平锁都是一样的逻辑，都需要处理重入问题：

```java

  public void unlock() {
        sync.release(1);
    }
    
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            // 如果当前线程不是独占锁的线程，则抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            // 只有当state为0时，才表示锁被释放
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

### 非公平锁

如上所述，公平锁和非公平锁的差别在于获取锁时是否遵守时间上的绝对顺序，公平锁总是将锁优先分配给队列中的队首线程，而非公平锁则是尝试分配给当前的线程：

```java
static final class NonfairSync extends Sync {
	 final void lock() {
	 // 优化策略，cas获取锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
        
      protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        
          final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
            // 没有判断是否存在前置节点
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
}
```

### 总结

ReentrantLock定义了公平和非公平两种同步器，其中的state表示持有锁线程的重入次数，同时也通过state维持锁的互斥，如果state不为0，表示锁已经被线程持有，否则当前线程可以尝试获取锁。




