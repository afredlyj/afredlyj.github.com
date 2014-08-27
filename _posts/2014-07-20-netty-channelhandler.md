---
layout: post  
title: Netty ChannelHandler处理流程分析
category: netty  
---

>前一段时间一直在看netty3源码，对Netty的启动过程的ChannelHandler处理流程有一定了解，现在记录下来，方便以后继续分析。


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

经过上面的分析可知，服务器每接收一个请求，就会调用一次`getPipeline`方法，生成一个新的pipeline，如果需要针对不同的请求，设定不同的pipeline，按理也可以做到，注意，这里的不同请求并不是只http协议中的不同请求url，解析url是HttpRequestDecoder的事情。

在Netty中，事件类型可以分为upstream ChannelEvent和downstream ChannelEvent，与之对应，ChannelHandler被分为UpstreamChannelHandler和DownStreamChannelHandler。