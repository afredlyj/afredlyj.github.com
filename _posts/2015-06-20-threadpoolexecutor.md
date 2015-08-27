---
layout: post
title: JAVA线程池源码分析
category: program
---

### ThreadPoolExecutor构造函数

ThreadPoolExecutor构造函数提供比较多的参数配置，方便开发者自定义：

~~~~~

public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
    
~~~~~

* corePoolSize  
线程池维持的线程数量，当然，`allowCoreThreadTimeOut`参数会对这个数量有影响。
* maximumPoolSize
线程池允许的最大线程数量。
* keepAliveTime
如果当前线程池的线程数量大于corePoolSize，keepAliveTime决定了多余空闲线程的最大存活时间
* workQueue
缓冲任务队列。
* threadFactory
线程工厂，用来创建新的线程。默认是`DefaultThreadFactory`。
* handler
线程池的拒绝策略。

如果在构造函数中没有指定threadFactory，默认会调用`Executors.defaultThreadFactory()`。默认的threadFactory，创建的所有线程属于同一个ThreadGroup，并且线程优先级相同，不是守护线程，也就是说，只要线程池没有退出，调用线程池的主线程也不会退出。根据默认的threadFactory，线程名称为`pool-#poolNumber#-#thread-threadNumber#`，可以通过jstack命令查看。

### 线程的生命周期

线程池只有在需要的时候才会创建新的线程。下面从`execute`函数入手分析，线程池中线程的生命周期：

~~~~

int c = ctl.get();
if (workerCountOf(c) < corePoolSize) {
  if (addWorker(command, true))
     return;
  c = ctl.get();
}
if (isRunning(c) && workQueue.offer(command)) {
int recheck = ctl.get();
if (! isRunning(recheck) && remove(command))
  reject(command);
else if (workerCountOf(recheck) == 0)
  addWorker(null, false);
}
else if (!addWorker(command, false))
  reject(command);

~~~~

主要流程如下：  
1. 如果当前worker数量小于配置的核心线程数量，则添加新的worker;  
2. 否则，判断线程池是否在运行，然后将提交的任务存放到任务队列中；  
3. 即使往任务队列添加成功，仍然需要double-check保证线程池正常运行；  
4. 如果添加任务队列失败，会尝试添加worker。  

#### addWorker

addWorker方法根据当前线程池的状态和设定的`core`或`maximum`判断是否需要创建新的woker。如果添加worker成功，则会启动worker对应的线程，并将`execute`方法中传入的任务作为线程的第一个任务。如果当前线程池状态不允许再添加新的worker，则该函数返回false。执行逻辑如下：

~~~~

private boolean addWorker(Runnable firstTask, boolean core) {
	retry:
	for (;;) {
		// 检查线程池状态
		
		for (;;) {
			// 检查线程数量是否超过指定容量
			
			// 增加worker数目，但是还没有真正添加worker
			if (compareAndIncrementWorkerCount(c)) {
				break retry:
			}
			if (runStateOf(c) != rs) {
				continue retry;
			}
			// CAS failed
		}
	}
	
	Worker w = new Worker(firstTask);
	mainLock.lock;
	try {
		workers.add(w)
	} finally {
		mainLock.unlock;
	}
	
	w.thread.start();
	
	 if (! workerStarted)
        addWorkerFailed(w);
	
}

~~~~

添加worker数量和创建新的worker，并将worker添加到woker set中这两个过程并没有同时加锁，在函数的最后，如果woker添加失败，线程池会加锁回滚。线程池将任务提交和任务执行解藕，addWoker分析完成之后，任务提交模块的功能完成了，如果一切顺利，此时已经成功将任务提交给线程池。

Worker是`Runnable`接口的子类，有两个主要的成员变量：`firstTask`和`thread`，分别代表任务和线程对象。在添加worker时，由上文的`addWorker`方法可知，worker的绑定线程也随之启动，接下来分析线程的执行，Worker类的run方法大致如下： 

~~~~

public void run() {
	Runnable task = this.firstTask;
	while (task != null || (task = getTask()) != null){
		beforeExecute();
		task.run();
		afterExecute();
	}
	
	processWorkerExit();
}

~~~~

也就是说，worker的绑定线程启动之后，只要任务队列有数据，就会不停的跑任务。`beforeExecute`和`afterExecute`是两个钩子方法，默认没有添加任何逻辑，如果实现自己的`ThreadPoolExecutor`，可以重写这两个方法，实现一些统计分析逻辑。

线程池中的线程是可以回收的，有两个控制变量`allowCoreThreadTimeOut`和`keepAliveTime`，接下来看看线程池是怎么控制线程的回收的。主要是`getTask`和`processWorkerExit`两个方法。

#### getTask

getTask方法就是一个无限循环，退出有几种可能：  
1. 线程池当前是退出状态，则直接返回null  
2. 成功取到任务  
3. 超时退出  
  
从任务队列中取任务的部分代码如下：

~~~~

timed = allowCoreThreadTimeOut || wc > corePoolSize;
try {
     Runnable r = timed ?
            workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
     if (r != null)
           return r;
     timedOut = true;
     } catch (InterruptedException retry) {
         timedOut = false;
     }
     
~~~~  

首先判断取任务时需要超时设置，如果配置了`allowCoreThreadTimeOut`或者当前线程数量大于配置的`corePoolSize`，则取任务时最多等待`keepAliveTime`，否则一直阻塞等待，成功取到任务则直接返回，否则此次循环超时，继续下一轮循环。`getTask`方法使用CAS避免加锁操作，提高线程池的并发性能。

#### processWorkerExit

processWorkerExit方法是线程退出前执行的方法，执行完成之后，线程的run函数退出，整个线程退出。线程退出while循环，可能是正常退出，也可能是异常中断，所以需要分开处理。如果线程是正常退出，并且线程不够用，整个方法会调用`addWorker`补充线程。

### 线程池的使用

#### execute vs submit (TODO)

### 参考  

1. [线程池数据结构与线程构造方法](http://www.blogjava.net/xylz/archive/2011/01/18/343183.html)
1. [ThreadPoolExecutor thread safe](http://stackoverflow.com/questions/1702386/is-threadpoolexecutor-thread-safe)