---
layout: post  
title: 了解Netyy线程模型  
category: netty  
---
### netty 线程模型

 * netty  boss和workder线程池使用`Executors.newCachedThreadPool()`，怎么固定线程数量？  
 以NIO服务端为例，在`ServerBootstrap`初始化时，会指定boss和worker的线程池，并初始化boss和worker实例，默认情况下，boss数量为1，worker数量为cpu * 2，boss和worker都是`Runnable`对象，就以`NioServerBoss`为例，并结合netty服务器的启动过程分析。
 首先，boss对象池的构造函数：  
 
 ~~~~  
   public NioServerBossPool(Executor bossExecutor, int bossCount, ThreadNameDeterminer determiner) {  
        super(bossExecutor, bossCount);
        this.determiner = determiner;
    }

    /**
     * Create a new instance using no {@link ThreadNameDeterminer}
     *
     * @param bossExecutor  the {@link Executor} to use for server the {@link NioServerBoss}
     * @param bossCount     the number of {@link NioServerBoss} instances this {@link NioServerBossPool} will hold
     */
    public NioServerBossPool(Executor bossExecutor, int bossCount) {
        this(bossExecutor, bossCount, null);
    }

    @Override
    protected NioServerBoss newBoss(Executor executor) {
        return new NioServerBoss(executor, determiner);
    }
 ~~~~  
 
 `newBoss`用来创建新的boss对象，NioServerBossPool初始化过程中，会初始化指定的bossCount个boss对象，这默认bossCount等于1，根据[java doc](http://netty.io/3.6/api/index.html)继承类图，可以追踪boss对象的初始化：  
 
 ~~~~  
     AbstractNioSelector(Executor executor, ThreadNameDeterminer determiner) {
        this.executor = executor;
        openSelector(determiner);
    }
    
    /**
     * Start the {@link AbstractNioWorker} and return the {@link Selector} that will be used for
     * the {@link AbstractNioChannel}'s when they get registered
     */
    private void openSelector(ThreadNameDeterminer determiner) {

        logger.info("open selector : " + Thread.currentThread().getName());

        try {
            selector = Selector.open();
        } catch (Throwable t) {
            throw new ChannelException("Failed to create a selector.", t);
        }

        // Start the worker thread with the new Selector.
        boolean success = false;
        try {
            DeadLockProofWorker.start(executor, newThreadRenamingRunnable(id, determiner));
            success = true;
        } finally {
            if (!success) {
                // Release the Selector if the execution fails.
                try {
                    selector.close();
                } catch (Throwable t) {
                    logger.warn("Failed to close a selector.", t);
                }
                selector = null;
                // The method will return to the caller at this point.
            }
        }
        assert selector != null && selector.isOpen();
    }
 ~~~~  
 
 到这里就会发现，创建每个boss对象时，会将该`Runnable`对象（boss对象和worker对象都是Runnable对象），放到executor中执行：  
 
 ~~~~  
 public static void start(final Executor parent, final Runnable runnable) {
        if (parent == null) {
            throw new NullPointerException("parent");
        }
        if (runnable == null) {
            throw new NullPointerException("runnable");
        }

        logger.debug("parent class : {}", parent.getClass());

        parent.execute(new Runnable() {
            public void run() {
                PARENT.set(parent);
                try {
                    runnable.run();
                } finally {
                    PARENT.remove();
                }
            }
        });
    }
 ~~~~
 
 提交任务之后，`ThreadPoolExecutor`线程池或从池中取出一个空闲线程，或创建一个新的线程执行指定任务。至此，每个boss或worker对象，都是从线程池中取出一个线程，执行相关逻辑。  
 netty线程模型的核心代码在`AbstractNioSelecor`中，这块逻辑，连同Reactor线程模型，都不了解，还有待继续努力。  
 
   
   