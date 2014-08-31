---
layout: post  
title: Netty ChannelHandler处理流程分析
category: netty  
---

>前一段时间一直在看netty3源码，对Netty的启动过程的ChannelHandler处理流程有一定了解，现在记录下来，方便以后继续分析。

### boss和worker的任务队列

ChannelHandler采用了职责链设计模式，Channel上的事件根据ChannelHandler的注册顺序依次执行。在Netty中，相邻的两个ChannelHandler并不直接交互，而是交给ChannelPipeline和ChannelHandlerContext，一般情况下，在服务器或者客户端启动时，需要调用BootStrap的`setPipelineFactory`方法，设置ChannelPipelineFactory，比如netty example就有如下示例代码：  

~~~~  

   
    public class HttpSnoopServerPipelineFactory implements ChannelPipelineFactory {
        public ChannelPipeline getPipeline() throws Exception {
        // Create a default pipeline implementation.
        ChannelPipeline pipeline = pipeline();

        // Uncomment the following line if you want HTTPS
        //SSLEngine engine = SecureChatSslContextFactory.getServerContext().createSSLEngine();
        //engine.setUseClientMode(false);
        //pipeline.addLast("ssl", new SslHandler(engine));

        pipeline.addLast("decoder", new HttpRequestDecoder());
        // Uncomment the following line if you don't want to handle HttpChunks.
        //pipeline.addLast("aggregator", new HttpChunkAggregator(1048576));
        pipeline.addLast("encoder", new HttpResponseEncoder());
        // Remove the following line if you don't want automatic content compression.
        pipeline.addLast("deflater", new HttpContentCompressor());
        pipeline.addLast("handler", new HttpSnoopServerHandler());
        return pipeline;
    }
}  
~~~~  

`getPipeline`方法会在boss线程accept连接时被调用： 

~~~~
    
// NioServerBoss.java

    private static void registerAcceptedChannel(NioServerSocketChannel parent, SocketChannel acceptedSocket,
                                         Thread currentThread) {
        try {
            ChannelSink sink = parent.getPipeline().getSink();
            ChannelPipeline pipeline =
                    parent.getConfig().getPipelineFactory().getPipeline();
            log.debug("registerAcceptedChannel get a new pipeline");
            NioWorker worker = parent.workerPool.nextWorker();
            worker.register(new NioAcceptedSocketChannel(
                    parent.getFactory(), pipeline, parent, sink
                    , acceptedSocket,
                    worker, currentThread), null);
        } catch (Exception e) {
            if (logger.isWarnEnabled()) {
                logger.warn(
                        "Failed to initialize an accepted socket.", e);
            }

            try {
                acceptedSocket.close();
            } catch (IOException e2) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Failed to close a partially accepted socket.",
                            e2);
                }
            }
        }
    }
~~~~

随后，boss线程会将该Channel注册到worker线程的任务队列中，交给worker线程处理，这是Netty Reactor线程模型的一部分，具体的处理流程我还不是很清楚，关于Reactor模式，更多信息可以访问[这里](http://www.cs.wustl.edu/~schmidt/PDF/reactor-siemens.pdf
)。

`NioWorker.register`方法，往worker线程任务队列中添加一个任务，任务的run方法如下：  

~~~~

      public void run() {
        	
        	log.debug("RegisterTask run ... ");
        	
            SocketAddress localAddress = channel.getLocalAddress();
            SocketAddress remoteAddress = channel.getRemoteAddress();

            if (localAddress == null || remoteAddress == null) {
                if (future != null) {
                    future.setFailure(new ClosedChannelException());
                }
                close(channel, succeededFuture(channel));
                return;
            }

            try {
                if (server) {
                    channel.channel.configureBlocking(false);
                }

                channel.channel.register(
                        selector, channel.getRawInterestOps(), channel);

                if (future != null) {
                    channel.setConnected();
                    future.setSuccess();
                }

                if (server || !((NioClientSocketChannel) channel).boundManually) {
                	log.debug("trying to bound ... ");
                    fireChannelBound(channel, localAddress);
                }
                fireChannelConnected(channel, remoteAddress);
            } catch (IOException e) {
                if (future != null) {
                    future.setFailure(e);
                }
                close(channel, succeededFuture(channel));
                if (!(e instanceof ClosedChannelException)) {
                    throw new ChannelException(
                            "Failed to register a socket to the selector.", e);
                }
            }
        }

~~~~


### ChannelPipeline 职责链
 
调用`Channels.fireChannelConnected`方法，往下走之前，有必要梳理一下Pipeline中的职责链。回到类`HttpSnoopServerPipelineFactory`，这个类实现了`ChannelPipelineFactory`
，对每个新生成的channel，都会调用这个ChannelPipelineFactory的`getPipeline`返回新的Pipeline，Pipeline是ChannelHandler的集合，在Netty中，Channel是通信的载体，每个ChannelHandler负责具体的业务逻辑，比如通信协议的编解码，请求数据的处理等等，ChannelHandler要注册到ChannelPipeline中才会起作用，注册的方式有几种，对应ChannlePipeline的几种方法，这里只说明一下最常用的'addLast'：

~~~~  

// DefaultChannelPipeline.java  

    public synchronized void addLast(String name, ChannelHandler handler) {
        if (name2ctx.isEmpty()) {
            init(name, handler);
        } else {
            checkDuplicateName(name);
            DefaultChannelHandlerContext oldTail = tail;
            DefaultChannelHandlerContext newTail = new DefaultChannelHandlerContext(oldTail, null, name, handler);

            callBeforeAdd(newTail);

            oldTail.next = newTail;
            tail = newTail;
            name2ctx.put(name, newTail);
            logger.info("" + name2ctx);
            callAfterAdd(newTail);
        }
    }

~~~~ 

每个ChannelPipeline都维护一个ChannelHandlerContext的map，以及两个ChannelHandlerContext的首尾引用，当map为空时，ChannelHandler链为空，初始化时，将首尾引用同时指向链中的唯一一个ChannelHandlerContext。可以看看ChannelHandlerContext默认实现的相关属性：

~~~~  

// DefaultChannelHandlerContext.java

        volatile DefaultChannelHandlerContext next;
        volatile DefaultChannelHandlerContext prev;
        private final String name;
        private final ChannelHandler handler;
        private final boolean canHandleUpstream;
        private final boolean canHandleDownstream;
        private volatile Object attachment;

        DefaultChannelHandlerContext(
                DefaultChannelHandlerContext prev, DefaultChannelHandlerContext next,
                String name, ChannelHandler handler) {

            if (name == null) {
                throw new NullPointerException("name");
            }
            if (handler == null) {
                throw new NullPointerException("handler");
            }
            canHandleUpstream = handler instanceof ChannelUpstreamHandler;
            canHandleDownstream = handler instanceof ChannelDownstreamHandler;

            if (!canHandleUpstream && !canHandleDownstream) {
                throw new IllegalArgumentException(
                        "handler must be either " +
                        ChannelUpstreamHandler.class.getName() + " or " +
                        ChannelDownstreamHandler.class.getName() + '.');
            }

            this.prev = prev;
            this.next = next;
            this.name = name;
            this.handler = handler;
        }

~~~~ 
 
由于Pipeline的职责链可以动态改变，所以一个Context的前后引用被设定为volatile，通过使用`prev` 和 `next`引用，整个Pipline形成了一个双向链表，结合之前的`head` 和 `tail` 首尾引用，可以快速在不同的ChannelHandler中移动。

`addLast`方法将当前的ChannelHandler存放到 `name2ctx` 映射map中，并添加到ChannelPipline末尾，如此一来，HttpSnoopServerPipelineFactory中生成的pipeline，双向链表简略图如下：  

~~~~  

decoder(head) <----> aggregator <----> deflater <----> handler(tail)

~~~~

### ChannelPipeline 处理流程

继续看Channles的fireChannelConnected方法：

~~~~  

    public static void fireChannelConnected(Channel channel, SocketAddress remoteAddress) {
    	log.debug("worker's channel connnected : {} , going to call pipeline sendUpstream", channel.getPipeline().getNames());
        channel.getPipeline().sendUpstream(
                new UpstreamChannelStateEvent(
                        channel, ChannelState.CONNECTED, remoteAddress));
    }

~~~~  

调用了`DefaultChannlePipeline.sendUpstream` ：

~~~~  


    public void sendUpstream(ChannelEvent e) {
    	log.debug("pipeline head : {}, {}", this.head.getName(), this);
        DefaultChannelHandlerContext head = getActualUpstreamContext(this.head);
        if (head == null) {
            if (logger.isWarnEnabled()) {
                logger.warn(
                        "The pipeline contains no upstream handlers; discarding: " + e);
            }

            return;
        }

        sendUpstream(head, e);
    }

    void sendUpstream(DefaultChannelHandlerContext ctx, ChannelEvent e) {
        try {
            ((ChannelUpstreamHandler) ctx.getHandler()).handleUpstream(ctx, e);
        } catch (Throwable t) {
            notifyHandlerException(e, t);
        }
    }

    private DefaultChannelHandlerContext getActualUpstreamContext(DefaultChannelHandlerContext ctx) {
        if (ctx == null) {
            return null;
        }

        DefaultChannelHandlerContext realCtx = ctx;
        log.debug("realCtx : {}, {}", realCtx.getName(), realCtx.getHandler().getClass());
        while (!realCtx.canHandleUpstream()) {
            realCtx = realCtx.next;
            if (realCtx == null) {
                return null;
            }
        }

        return realCtx;
    }

~~~~

在这里看到了Pipeline中的head，从head引用开始寻找，找到第一个UpStreamChannleHandler，如果找不到，返回`null`，上文中的`HttpRequestDecoder`是SimpleChannelUpstreamHandler的子类， handleUpstream实现如下 : 

~~~~  

    public void handleUpstream(
            ChannelHandlerContext ctx, ChannelEvent e) throws Exception {

    	log.info("SimpleChannelUpstreamHandler handleUpstream : {}, Thread : {}", e.getClass(), Thread.currentThread().getId());
        if (e instanceof MessageEvent) {
            messageReceived(ctx, (MessageEvent) e);
        } else if (e instanceof WriteCompletionEvent) {
            WriteCompletionEvent evt = (WriteCompletionEvent) e;
            writeComplete(ctx, evt);
        } else if (e instanceof ChildChannelStateEvent) {
            ChildChannelStateEvent evt = (ChildChannelStateEvent) e;
            if (evt.getChildChannel().isOpen()) {
                childChannelOpen(ctx, evt);
            } else {
                childChannelClosed(ctx, evt);
            }
        } else if (e instanceof ChannelStateEvent) {
            ChannelStateEvent evt = (ChannelStateEvent) e;
            log.info("ChannelStateEvent : {}, threadId : {}", ((ChannelStateEvent) e).getState(), Thread.currentThread().getId());
            switch (evt.getState()) {
            case OPEN:
                if (Boolean.TRUE.equals(evt.getValue())) {
                    channelOpen(ctx, evt);
                } else {
                    channelClosed(ctx, evt);
                }
                break;
            case BOUND:
                if (evt.getValue() != null) {
                    channelBound(ctx, evt);
                } else {
                    channelUnbound(ctx, evt);
                }
                break;
            case CONNECTED:
                if (evt.getValue() != null) {
                    channelConnected(ctx, evt);
                } else {
                    channelDisconnected(ctx, evt);
                }
                break;
            case INTEREST_OPS:
                channelInterestChanged(ctx, evt);
                break;
            default:
                ctx.sendUpstream(e);
            }
        } else if (e instanceof ExceptionEvent) {
            exceptionCaught(ctx, (ExceptionEvent) e);
        } else {
            ctx.sendUpstream(e);
        }
    }

~~~~

Netty pipeline的整个处理流程，就是在不停的ChannelHandler中调整，根据源码可知，默认情况下，ChannelHanlder中的业务逻辑都是在Netty的io线程（由执行boss和workder的线程组成）中完成的，更准确的说，是在worker执行线程中跑的，对于一般的编解码操作，建议直接在io线程中处理，如果ChannelHandler业务逻辑复杂，或者比较耗时，可以另开线程，主要考虑到线程上下文切换的开销。

经过上面的分析可知，服务器每接收一个请求，就会调用一次`getPipeline`方法，生成一个新的pipeline，如果需要针对不同的请求，设定不同的pipeline，按理也可以做到，注意，这里的不同请求并不是指http协议中的不同请求url，解析url是HttpRequestDecoder的事情。
