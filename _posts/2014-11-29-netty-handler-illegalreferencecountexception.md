---
layout: post  
title: netty自定义handler异常 IllegalReferenceCountException
category: bug  
---

最近看netty4的源码，自定义ChannelHandler，平时都是在pipeline最后添加一个Decoder完事，后来想想可以尝试将验证请求这部分功能添加到handler中，才引发下文提到的bug。

#### 异常复现  

pipeline初始化代码如下：

~~~~  
    logger.debug("init channel");
    ChannelPipeline pipeline = ch.pipeline();
    pipeline.addLast("decoder", new HttpRequestDecoder());
    pipeline.addLast("encoder", new HttpResponseEncoder());
    /**
     * Be aware that you need to have the HttpResponseEncoder or HttpRequestEncoder
     * before the HttpObjectAggregator in the ChannelPipeline.
     */
    pipeline.addLast("aggregator", new HttpObjectAggregator(1048576));
    pipeline.addLast("checkSum", new CheckSumHandler());
    if (bzGroup != null) {
        pipeline.addLast(bzGroup, "serverHandler", new HttpServerHandler());
    } else {
        pipeline.addLast("serverHandler", new HttpServerHandler());
    }

~~~~

这里自定义两个handler，CheckSumHandler和HttpServerHandler，CheckSumHandler部分代码如下：

~~~~

// CheckSumHandler.java
@Override
    protected void decode(ChannelHandlerContext ctx, FullHttpRequest msg, List<Object> out) throws Exception {

        logger.debug("decode request");

        if (!msg.getDecoderResult().isSuccess()) {
            logger.warn("decode failure");
            sendError(ctx, HttpResponseStatus.BAD_REQUEST);
            return;
        }

        // check sum
        if (!checkSum(msg)) {
            logger.warn("check sum failure");
            sendError(ctx, HttpResponseStatus.BAD_REQUEST);
            return;
        }

        logger.debug("decode success");
        out.add(msg);
    }

~~~~

CheckSumHandler是`MessageToMessageDecoder`的子类，实现`decode`方法，CheckSumHandler会对请求数据验证，如果非法，直接拒绝，返回404，否则继续下面的业务落后，执行HttpServerHandler，HttpServerHandler是`SimpleChannelInboundHandler`的子类，这里只是回写字符串，代码不需要展示。

这段代码如果checkSum失败，服务端可以正常运行，但是如果checkSum通过，在HttpServerHandler中会有如下异常：

~~~~    

10:08:09.834 [nioEventLoopGroup-3-1] ERROR a.d.h.s.handler.HttpServerHandler 87 - http server handler exception
io.netty.util.IllegalReferenceCountException: refCnt: 0, decrement: 1
	at io.netty.buffer.AbstractReferenceCountedByteBuf.release(AbstractReferenceCountedByteBuf.java:102)

~~~~


#### 异常分析

通过逐步debug，发现HttpServerHandler报异常的代码在`getBytes`方法中：  

~~~~  

 ByteBuf content = request.content();
 int readableBytes = content.readableBytes();
 byte[] bs = null;
 bs = new byte[readableBytes];
 content.getBytes(0, bs, 0, readableBytes); 
~~~~

这里的content实现为`CompositeByteBuf`，查看源码，追踪下面的方法：

~~~~

/**
 * Should be called by every method that tries to access the buffers content to check
 * if the buffer was released before.
 */
protected final void ensureAccessible() {
    if (refCnt() == 0) {
        throw new IllegalReferenceCountException(0);
    }
 }

~~~~

在netty4中，对象的生命周期由引用计数器控制，ByteBuf就是如此，每个对象的初始化引用计数为1，调用一次release方法，引用计数器会减1，当尝试访问计数器为0的，对象时，会抛出`IllegalReferenceCountException`，正如`ensureAccessible`的实现，更加详细的解释可以参考[官方文档](http://netty.io/wiki/reference-counted-objects.html)

回到CheckSumHandler的父类`MessageToMessageDecoder`，它会在decode之后release：

~~~~  

if (acceptInboundMessage(msg)) {
    @SuppressWarnings("unchecked")
    I cast = (I) msg;
    try {
        decode(ctx, cast, out);
    } finally {
        ReferenceCountUtil.release(cast);
    }
 } else {
    out.add(msg);
 }  
~~~~

所以需要在CheckSumHandler中将引用计数加1，根据谁最后使用，谁负责释放的原则，HttpServerHandler会负责调用release方法，这部分功能直接由父类`SimpleChannelInboundHandler`实现，更改后的代码如下：

~~~~  

@Override
    protected void decode(ChannelHandlerContext ctx, FullHttpRequest msg, List<Object> out) throws Exception {

        logger.debug("decode request");

        if (!msg.getDecoderResult().isSuccess()) {
            logger.warn("decode failure");
            sendError(ctx, HttpResponseStatus.BAD_REQUEST);
            return;
        }

        // check sum
        if (!checkSum(msg)) {
            logger.warn("check sum failure");
            sendError(ctx, HttpResponseStatus.BAD_REQUEST);
            return;
        }

        /**
         *  Be aware that you need to call {@link io.netty.util.ReferenceCounted#retain()} on messages that are just passed through if they
         * are of type {@link io.netty.util.ReferenceCounted}. This is needed as the {@link MessageToMessageDecoder} will call
         * {@link io.netty.util.ReferenceCounted#release()} on decoded messages.
         */
        ReferenceCountUtil.retain(msg);
        logger.debug("decode success");
        out.add(msg);
    }
~~~~  

英文注释部分为`MessageToMessageDecoder`的javadoc，netty的文档真心不错，泪奔~
