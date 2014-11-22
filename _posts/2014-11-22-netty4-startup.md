## netty 4 启动流程分析

>根据EchoServer初始化代码，跟踪netty 4的初始化


整个过程比较复杂，会牵涉到很多分支，根据初始化流程分为下面几个部分：

 * [NioServerSocketChannel 初始化](#nioserversocketchannel)
 * [NioEventLoopGroup 初始化](#nioeventloopgroup)
 * [ServerBootstrap 端口绑定](#serverbootstrap)
 * [pipeline 处理逻辑](#pipeline)
 
 
其中，NioServerSocketChannel和NioEventLoopGroup的初始化，都是在ServerBootstrap接口绑定过程中完成的，为了方便理清流程，这里分开分析。

### <span id="nioserversocketchannel">NioServerSocketChannel 初始化</span>

ServerBootstrap.channel()方法创建一个新的ChannelFactory，BootstrapChannelFactory

~~~~  
   public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new BootstrapChannelFactory<C>(channelClass));
    }  
~~~~  

`BootstrapChannelFactory` 实现如下：

~~~~ 
     
     private static final class BootstrapChannelFactory<T extends Channel> implements ChannelFactory<T> {
        private final Class<? extends T> clazz;

        BootstrapChannelFactory(Class<? extends T> clazz) {
            this.clazz = clazz;
        }

        @Override
        public T newChannel() {
            try {
                return clazz.newInstance();
            } catch (Throwable t) {
                throw new ChannelException("Unable to create Channel from class " + clazz, t);
            }
        }

        @Override
        public String toString() {
            return clazz.getSimpleName() + ".class";
        }
    }
~~~~  

调用了`NioServerSocketChannel`的默认构造函数：

~~~~  
   public NioServerSocketChannel() {
        super(null, newSocket(), SelectionKey.OP_ACCEPT);
        config = new DefaultServerSocketChannelConfig(this, javaChannel().socket());
    }
~~~~ 

该类的继承关系可以从netty [javadoc](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html)了解到，最后在`AbstractChannel`中创建了一个`DefaultChannelPipeline`对象：

~~~~  

    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        unsafe = newUnsafe();
        pipeline = new DefaultChannelPipeline(this);
    }

~~~~  

这里的`newUnsafe()`方法则由`AbstractChannel`子类`AbstractNioMessageChannel`实现，返回一个`NioMessageUnsafe`对象。

~~~~  
    @Override
    protected AbstractNioUnsafe newUnsafe() {
        return new NioMessageUnsafe();
    }
~~~~

>todo : netty 4中 `Unsafe` 的作用是什么？

再来看看netty4 中的`DefaultChannelPipe`和n`DefaultChannelHandlerContext`初始化，和netty3 将pipeline的创建交给用户不同，netty4 将pipeline的创建权限回收，由框架自行创建（在AbstractChannel构造函数中创建），用户只需要使用即可。

~~~~
//DefaultChannelPipeline.java
    public DefaultChannelPipeline(Channel channel) {
        if (channel == null) {
            throw new NullPointerException("channel");
        }
        this.channel = channel;

        TailHandler tailHandler = new TailHandler();
        tail = new DefaultChannelHandlerContext(this, null, generateName(tailHandler), tailHandler);

        HeadHandler headHandler = new HeadHandler(channel.unsafe());
        head = new DefaultChannelHandlerContext(this, null, generateName(headHandler), headHandler);

        head.next = tail;
        tail.prev = head;
    }
    
//DefaultChannelHandlerContext.java
@SuppressWarnings("unchecked")
    DefaultChannelHandlerContext(
            DefaultChannelPipeline pipeline, EventExecutorGroup group, String name, ChannelHandler handler) {

        if (name == null) {
            throw new NullPointerException("name");
        }
        if (handler == null) {
            throw new NullPointerException("handler");
        }

        channel = pipeline.channel;
        this.pipeline = pipeline;
        this.name = name;
        this.handler = handler;

        if (group != null) {
            // Pin one of the child executors once and remember it so that the same child executor
            // is used to fire events for the same channel.
            EventExecutor childExecutor = pipeline.childExecutors.get(group);
            if (childExecutor == null) {
                childExecutor = group.next();
                pipeline.childExecutors.put(group, childExecutor);
            }
            executor = childExecutor;
        } else {
            executor = null;
        }
    }   
~~~~ 

netty初始化时，`public DefaultChannelPipeline(Channel channel)`  传入的是一个`NioServerSocketChannel`对象。pipeline初始化完成之后，会在职责链中添加两个默认的handler : `HeadHandler` 和 `TailHandler` ，分别继承于 `ChannelOutboundHandler`和`ChannelInboundHandler`。 
另外需要注意的是，pipeline构造函数创建新的context时，将group设为null。

>todo : EventExecutor和EventExecutorGroup的区别？


### <span id="nioeventloopgroup"]>NioEventLoopGroup 初始化</span>

NioEventLoopGroup初始化主要任务是创建线程池和线程，并且创建任务队列。

默认情况下，线程池中线程的数量是cpu * 2，具体代码如下：

~~~~  
static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }

~~~~  

接下来看线程池初始化的核心代码：

~~~~  
protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (threadFactory == null) {
            threadFactory = newDefaultThreadFactory();
        }

        children = new SingleThreadEventExecutor[nThreads];
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(threadFactory, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }
    }
~~~~

如果参数threadFactory为空，则创建一个默认的DefaultThreadFactory。接下来创建指定数量nThreads个SingleThreadEventExecutor对象，其中`newChild`方法由NioEventLoopGroup实现：

~~~~  
@Override
    protected EventExecutor newChild(
            ThreadFactory threadFactory, Object... args) throws Exception {
        return new NioEventLoop(this, threadFactory, (SelectorProvider) args[0]);
    }
~~~~

NioEventLoop继承关系可参考[官方文档](http://netty.io/4.0/api/io/netty/channel/nio/NioEventLoop.html)，父类SingleThreadEventExecutor调用threadFactory创建新的线程，并指定任务，在指定的匿名任务类中，调用了自身的抽象方法`run`，由子类NioEventLoop实现，实现的是一个无限循环的处理逻辑。每一次循环，都会尝试将任务队列的任务全部执行，直到执行时间超过指定时间。

NioEventLoop的主要任务：

0. 负责I/O读写
1. 系统Task
调用NioEventLoop.execute(Runnable task)
2. 定时任务
调用NioEventLoop.schedule(Runnable task, long delay, TimeUnit unit)


初始化时，taskQueue什么时候添加任务？
EventLoop调用execute时，见下文`AbstractUnsafe.register`代码。


最后创建一个新的任务队列。在父类SingleThreadEventExecutor中，任务队列的默认实现是`LinkedBlockingQueue`，子类NioEventLoop重写了newTaskQueue方法，任务队列是`ConcurrentLinkedQueue`。

NioEventLoopGroup初始化后，并没有启动线程执行任务队列，而是在端口绑定过程中启动线程，详情可参考下文。


### <span id="serverbootstrap">ServerBootstrap 端口绑定</span>

端口绑定时会创建新的NioServerSocketChannel，并将其注册到任务队列，这部分根据`doBind`的调用逻辑分方法分析。

~~~~  
//AbstractBootstrap.java
private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regPromise = initAndRegister();
        final Channel channel = regPromise.channel();
        final ChannelPromise promise = channel.newPromise();
        if (regPromise.isDone()) {
            doBind0(regPromise, channel, localAddress, promise);
        } else {
            regPromise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    doBind0(future, channel, localAddress, promise);
                }
            });
        }

        return promise;
    }
    
~~~~  

#### initAndRegister方法

首先，设置channel的初始化属性值，并在当前channel的pipeline中添加`ServerBootstrapAcceptor` 对象，至于具体的添加过程，需要参考下文ChannelInitializer的处理过程。

~~~~  
//ServerBootstrap.java
@Override
    void init(Channel channel) throws Exception {
        final Map<ChannelOption<?>, Object> options = options();
        synchronized (options) {
            channel.config().setOptions(options);
        }

        final Map<AttributeKey<?>, Object> attrs = attrs();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }

        ChannelPipeline p = channel.pipeline();
        if (handler() != null) {
            p.addLast(handler());
        }

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(Channel ch) throws Exception {
                ch.pipeline().addLast(new ServerBootstrapAcceptor(
                        currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }    
~~~~

在`ServerBootstrap.init`中，向pipeline中添加了一个`ChannelInitializer`，此时的pipeline在初始化时，已经有两个handler：`HeadHandler`和`TailHandler`，所以当前的pipeline双向链表结构为： 

~~~~  
HeadHandler <--> channelInitializer <--> TailHandler  
~~~~  

`initAndRegister`方法创建新的Channel对象，初始化（所以这里`init`的channel是NioServerSocketChannel），并将该channel注册到处理线程池中。最后返回一个DefaultChannelPromise对象。

~~~~  
      final ChannelFuture initAndRegister() {
        final Channel channel = channelFactory().newChannel();
        try {
            init(channel);
        } catch (Throwable t) {
            channel.unsafe().closeForcibly();
            return channel.newFailedFuture(t);
        }

        ChannelPromise regPromise = channel.newPromise();
        group().register(channel, regPromise);
        if (regPromise.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now beause the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regPromise;
    }
~~~~

channel初始化之后（init方法），回到`initAndRegister`方法，继而调用了`MultithreadEventLoopGroup.register`：

~~~~  
// MultithreadEventExecutorGroup.java
@Override
    public EventExecutor next() {
        return children[Math.abs(childIndex.getAndIncrement() % children.length)];
    }

// MultithreadEventLoopGroup.java
    @Override
    public ChannelFuture register(Channel channel, ChannelPromise promise) {
        return next().register(channel, promise);
    }

    @Override
    public EventLoop next() {
        return (EventLoop) super.next();
    }

~~~~

>todo : 每次都是按顺序取EventExecutor，会不会有性能问题？

`next()`方法返回一个`NioEventLoop`对象，是`SingleThreadEventLoop`的子类：

~~~~

//SingleThreadEventLoop.java

@Override
    public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
        if (channel == null) {
            throw new NullPointerException("channel");
        }
        if (promise == null) {
            throw new NullPointerException("promise");
        }

        channel.unsafe().register(this, promise);
        return promise;
    }

~~~~

根据上文，这里的channel是一个NioServerSocketChannel对象，unsafe为一个NioMessageUnsafe对象：

~~~~

// AbstractChannel.java  AbstractUnsafe
@Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was unaccepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    promise.setFailure(t);
                }
            }
        }

~~~~

这里的注册代码，会判断当前线程是否是线程池中的线程（`inEventLoop`），如果是，直接执行`register0`方法，否则，启动该eventLoop中的线程，并将`NioMessageUnsafe`中的注册任务添加到任务队列，在netty 4 中，正是通过这种方式保证数据安全，无须加锁，也减少了线程上下文切换。

~~~~  
// SingleThreadEventExecutor.java
@Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            startThread();
            addTask(task);
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp) {
            wakeup(inEventLoop);
        }
    }

~~~~  

回到`AbstractUnsafe.register` 方法，继续往下走，进入`register0`，继而调用`DefaultChannelPipeline.fireChannelRegistered`，该方法会从双向链表的头开始找，直到找到下一个`ChannelInboundHandler`，然后执行`invokeChannelRead`方法。在这里，最后会调用`ServerBootstrapAcceptor.channelRegistered`方法。

### <span id="pipeline">pipeline 处理逻辑</span>

~~~~   
// DefaultChannelPipeline.java
@Override
    public ChannelPipeline fireChannelRegistered() {
        head.fireChannelRegistered();
        return this;
    }

// DefaultChannelHandlerContext.java
@Override
    public ChannelHandlerContext fireChannelRegistered() {

        logger.info("pipeline names : {}", this.pipeline().names());

        final DefaultChannelHandlerContext next = findContextInbound();

        logger.info("find next in bound : {}", next.name());
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRegistered();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRegistered();
                }
            });
        }
        return this;
    }

    private void invokeChannelRegistered() {
        try {
            ((ChannelInboundHandler) handler()).channelRegistered(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    }

    private DefaultChannelHandlerContext findContextInbound() {
        DefaultChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!(ctx.handler() instanceof ChannelInboundHandler));
        return ctx;
    }
~~~~

看一下context中的executor从哪儿设置的:

~~~~

@Override
    public EventExecutor executor() {
        if (executor == null) {
            return channel().eventLoop();
        } else {
            return executor;
        }
    }
~~~~

由上文可知，DefaultChannelPipeline在初始化时，双向链表的head和tail并没有设置executor，查看NioServerSocketChannel的eventLoop()方法：

~~~~  

@Override
    public EventLoop eventLoop() {
        EventLoop eventLoop = this.eventLoop;
        if (eventLoop == null) {
            throw new IllegalStateException("channel not registered to an event loop");
        }
        return eventLoop;
    }
    
    
~~~~

构造函数并没有设置eventLoop成员变量，继续找，发现`AbstractUnsafe.register`中设置了eventLoop，根据上文ServerBootstrap的端口绑定发现，这里成功将一个NioEventLoop对象设置到AbstractChannel中。

#### ChannelInitializer的作用

netty 4 中， ChannelInitializer是ChannelInboundHandler的子类，作用是向pipeline中添加自定义的ChannelHandler，调用initchannel方法后，将自身从pipeline中删除，核心功能在ChannelInitializer的channelRegistered方法。

再次回到`DefaultChannelHandlerContext.fireChannelRegistered`，`findContextInbound`第一次调用会找到上文中`channelinitializer`所属的context。  

~~~~  
// DefaultChannelHandlerContext.java
private void invokeChannelRegistered() {
        try {
            ((ChannelInboundHandler) handler()).channelRegistered(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    }

~~~~  

来看看该`ChannelInitializer`对象的相关方法：

~~~~
@SuppressWarnings("unchecked")
    @Override
    public final void channelRegistered(ChannelHandlerContext ctx)
            throws Exception {
        boolean removed = false;
        boolean success = false;
        try {
            initChannel((C) ctx.channel());
            ctx.pipeline().remove(this);
            removed = true;
            ctx.fireChannelRegistered();
            success = true;
        } catch (Throwable t) {
            logger.warn("Failed to initialize a channel. Closing: " + ctx.channel(), t);
        } finally {
            if (!removed) {
                ctx.pipeline().remove(this);
            }
            if (!success) {
                ctx.close();
            }
        }
    }

~~~~

netty中的`ChannelInitializer`是抽象类，在这里回答了上文`ChannelInitializer.initChannel`调用问题，由上文ServerBootstrap可以找到实现：

~~~~  
p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(Channel ch) throws Exception {
        ch.pipeline().addLast(new ServerBootstrapAcceptor(
            currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        })

~~~~  

从源码可知，向`NioServerSocketChannel`中又添加了`ServerBootstrapAcceptor`对象。所以，到目前为止，该双向链表如下：

~~~~  
HeadHandler <--> channelInitializer <--> ServerBootstrapAcceptor <--> TailHandler  
~~~~  

回到抽象类ChannelInitializer，调用`pipeline.remove`方法，将该ChannelInitializer删除，继而又调用`ChannelHandlerContext.fireChannelRegistered`。

>todo : 根据`ServerBootstrapAcceptor`来看，直接跳过`chanelRead`方法了？

`TailHandler.channelRegistered`只是一个空函数，没有具体的操作。

到此为止，`AbstractUnsafe.register0`走到了`pipeline.fireChannelRegistered();`下一步。

`AbstractUnsafe.register0`走完。回到`AbstractBootstrap.doBind()`方法，调用`doBind0()`，最后调用`NioServerSocketChannel.doBind`方法，到此为止，服务端启动完成。

> 在调试过程中，发现pipeline中看不到HeadHandler，原因在于`names`和`toString`方法均将`head`跳过，详情可以参考`DefaultChannelPipeline`源码。


>回到上文，pipeline的处理逻辑，最后会调用handler.channelRead方法。暂时先分析到这里，至于怎么在各个handler中跳转，稍后再看。

> 数据读取操作什么时候启动，流程怎样的？

### netty 4 读取数据

#### NioEventLoop 任务线程执行

NioEventLoop 的执行流程，伪代码如下：

~~~~  

for (;;) {
    select();
    for (;;) {
       processSelectedKeys(); 
    }
    runAllTasks();
}  
~~~~

`processSelectedKeys`方法根据`readOps`判断是读还是写，服务端接受链接时，我们暂时只考虑读操作，读操作会调用`unsafe.read()`，对于`NioServerSocketChannel`，它的读操作就是接收客户端的TCP连接： 


~~~~

//AbstractNioUnsafe.java
try {
        for (;;) {
            int localRead = doReadMessages(readBuf);
            logger.info("local read : {}", localRead);
            if (localRead == 0) {
                break;
            }
            if (localRead < 0) {
                closed = true;
                break;
            }

            if (readBuf.size() >= maxMessagesPerRead | !autoRead) {
                break;
            }
        }
    } catch (Throwable t) {
        exception = t;
    }

  for (int i = 0; i < readBuf.size(); i ++) {
        pipeline.fireChannelRead(readBuf.get(i));
    }
~~~~  

接收连接之后，调用`pipeline.fireChannleRead`，试图从pipeline中找出第一个Inbound handler，然后调用`ChannelInboundHandler.channelRead`方法。

在上文分析ChannelInitializer作用过程中，可以看出，这里找到的是`ServerBootstrapAcceptor`，它的channelRead方法用来初始化`NioSocketChannel`相关参数，另外还初始化了pipeline，然后将该channel注册到childGroup（即subReactor）中，代码如下：

~~~~

@Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            Channel child = (Channel) msg;

            logger.debug("child channel class : {}", child.getClass());

            logger.info("ServerBootstrapAcceptor pipeline(just init by constructor) : {}", child.pipeline().names());

            logger.info("childHandler class : {}", childHandler.getClass());

            child.pipeline().addLast(childHandler);

            logger.info("after add child handler : {}", child.pipeline().names());

            for (Entry<ChannelOption<?>, Object> e: childOptions) {
                try {
                    if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                        logger.warn("Unknown channel option: " + e);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to set a channel option: " + child, t);
                }
            }

            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }

            try {
                childGroup.register(child);
            } catch (Throwable t) {
                child.unsafe().closeForcibly();
                logger.warn("Failed to register an accepted channel: " + child, t);
            }
        }

~~~~

从上面的代码得知，在该方法`childGroup.register`之前，所有的处理都是在`parantGroup`（即mainReactor）中。




