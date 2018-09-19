---
layout: blog
title: netty pipeline ChannelHandler 机制
date: 2018-09-15 22:58:02
categories: [netty]
tags: [netty]
---

> 每个channel内部都会持有一个`ChannelPipeline`对象，pipeline默认实现`DefaultChannelPipeline`的内部维护了一个`DefaultChannelHandlerContext`链表，如下图所示：

<!--more-->

{% asset_img 20180915113652503.png %}

当channel完成register、active、read等操作时，会触发pipeline的相应方法。
* 当channel注册到selector时，触发pipeline的fireChannelRegistered方法。
* 当channel的socket绑定完成时，触发pipeline的fireChannelActive方法。
* 当有客户端请求时，触发pipeline的fireChannelRead方法。
* 当响应客户端请求，pipeline执行完fireChannelRead，触发pipeline的fireChannelReadComplete方法（比如在fireChannelReadComplete方法中做flush操作）。

## 创建ChannelPipeline实例

在初始化一个NioSocketChannel流程中，会创建该channel默认的ChannelPipeline，也就是DefaultChannelPipeline。

```java
public class DefaultChannelPipeline implements ChannelPipeline { 
    // DefaultChannelHandlerContext保存了当前handler的上下文，如channel信息，默认实现head和tail。
    // head和tail是handler上下文
    final AbstractChannelHandlerContext head;
    final AbstractChannelHandlerContext tail;
    private final Channel channel;
 
    protected DefaultChannelPipeline(Channel channel) {
        this.channel = ObjectUtil.checkNotNull(channel, "channel");
        succeededFuture = new SucceededChannelFuture(channel, null);
        voidPromise =  new VoidChannelPromise(channel, true);
 
        tail = new TailContext(this);
        head = new HeadContext(this);
 
        head.next = tail;
        tail.prev = head;
    }
}
```

说到了DefaultChannelPipeline，就不得不提pipeline的事件传输机制（*inbound和outbound这两个操作*）。

### pipeline事件传输机制

ChannelHandlerContext保存当前channelHandler上下文， AbstractChannelHandlerContext 中有 inbound 和 outbound 两个 boolean 变量, 分别用于标识 Context 所对应的 handler 的类型, 即：
* inbound 为真时, 表示对应的 ChannelHandler 实现了 ChannelInboundHandler 方法.
* outbound 为真时, 表示对应的 ChannelHandler 实现了 ChannelOutboundHandler 方法.

inbound 和 outbound这两个字段有什么作用呢? 其实这还要从 ChannelPipeline 的传输的事件类型说起。Netty 的事件可以分为 Inbound 和 Outbound 事件。如下是从 Netty 官网上拷贝的一个图示:
```
                                          I/O Request
                                        via Channel or
                                    ChannelHandlerContext
                                                  |
+---------------------------------------------------+---------------+
|                           ChannelPipeline         |               |
|                                                  \|/              |
|    +---------------------+            +-----------+----------+    |
|    | Inbound Handler  N  |            | Outbound Handler  1  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
|               |                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  .               |
|               .                                   .               |
| ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
|        [ method call]                       [method call]         |
|               .                                   .               |
|               .                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
|               |                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler  1  |            | Outbound Handler  M  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
+---------------+-----------------------------------+---------------+
              |                                  \|/
+---------------+-----------------------------------+---------------+
|               |                                   |               |
|       [ Socket.read() ]                    [ Socket.write() ]     |
|                                                                   |
|  Netty Internal I/O Threads (Transport Implementation)            |
+-------------------------------------------------------------------+
```

从上图可以看出, inbound 事件和 outbound 事件的流向是不一样的, inbound 事件的流向是从下至上（` head -> customContext -> tail`）, 而 outbound 刚好相反, 是从上到下（`tail -> customContext -> head`）. 并且 inbound 的传递方式是通过调用相应的 `ChannelHandlerContext.fireIN_EVT()` 方法, 而 outbound 方法的的传递方式是通过调用 `ChannelHandlerContext.OUT_EVT() `方法. 例如 `ChannelHandlerContext.fireChannelRegistered()` 调用会发送一个 ChannelRegistered 的 inbound 给下一个ChannelHandlerContext, 而 ChannelHandlerContext.bind 调用会发送一个 bind 的 outbound 事件给 下一个 ChannelHandlerContext.

Inbound 事件传播方法有:
```java
ChannelHandlerContext.fireChannelRegistered()
ChannelHandlerContext.fireChannelActive()
ChannelHandlerContext.fireChannelRead(Object)
ChannelHandlerContext.fireChannelReadComplete()
ChannelHandlerContext.fireExceptionCaught(Throwable)
ChannelHandlerContext.fireUserEventTriggered(Object)
ChannelHandlerContext.fireChannelWritabilityChanged()
ChannelHandlerContext.fireChannelInactive()
ChannelHandlerContext.fireChannelUnregistered()
```

Oubound 事件传输方法有:
```java
ChannelHandlerContext.bind(SocketAddress, ChannelPromise)
ChannelHandlerContext.connect(SocketAddress, SocketAddress, ChannelPromise)
ChannelHandlerContext.write(Object, ChannelPromise)
ChannelHandlerContext.flush()
ChannelHandlerContext.read()
ChannelHandlerContext.disconnect(ChannelPromise)
ChannelHandlerContext.close(ChannelPromise)
```

注意, 如果我们捕获了一个事件, 并且想让这个事件继续传递下去, 那么需要调用 Context 相应的传播方法.例如:
```java
public class MyInboundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("Connected!");
        ctx.fireChannelActive();
    }
}

public clas MyOutboundHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void close(ChannelHandlerContext ctx, ChannelPromise promise) {
        System.out.println("Closing ..");
        ctx.close(promise);
    }
}
```
上面的例子中, MyInboundHandler 收到了一个 channelActive 事件, 它在处理后, 如果希望将事件继续传播下去, 那么需要接着调用 ctx.fireChannelActive().

inbound事件如何获取下一个ChannelHandlerContext，outbound事件如何获取下一个ChannelHandlerContext呢？它们分别使用下面两个方法来查找对应的下一个ChannelHandlerContext。
```java
private AbstractChannelHandlerContext findContextInbound() {
    AbstractChannelHandlerContext ctx = this;
    do {
        ctx = ctx.next;
    } while (!ctx.inbound);
    return ctx;
}
private AbstractChannelHandlerContext findContextOutbound() {
    AbstractChannelHandlerContext ctx = this;
    do {
        ctx = ctx.prev;
    } while (!ctx.outbound);
    return ctx;
}
```

ChannelPipeline、ChannelContext和ChannelHandler关系如下：
{% asset_img 20180916125556533.png %}

ChannelPipeline就是一个流水线，流水线上工人就是ChannelContext，每个ChannelContext有一个ChannelHandler。当事件（inbound或者outbound事件）在流水线上流转，也就是在各个ChannelContext传递过程中，如果是inbound事件(传播方向head -> tail)，Inbound 事件的处理者是 Channel, 如果用户没有实现自定义的处理方法, 那么Inbound 事件默认的处理者是 TailContext, 并且其处理方法是空实现；如果是outbound事件(传播方向tial -> head)，Outbound 事件的发起者是 Channel，Outbound 事件的处理者是Head中的unsafe。

### pipieline#addLast

往chanel.pipeline中添加ChannelHandler，一般是调用`ChannelPipeline#addLast`，示例如下（http server）：
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();

ServerBootstrap boot = new ServerBootstrap();
boot.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
		.localAddress(8080).childHandler(new ChannelInitializer<SocketChannel>() {
	@Override
	protected void initChannel(SocketChannel ch) throws Exception {
		ch.pipeline()
            // inbound
            .addLast("decoder", new HttpRequestDecoder())
            // inbound
            .addLast("aggregator", new HttpObjectAggregator(512 * 1024))
            // outbound
            .addLast("encoder", new HttpResponseEncoder())
            // inbound
            .addLast("handler", new HttpHandler());
	}
});
// ...
```

对于addLast(ChannelHandler)的添加，应该以什么顺序呢？我们知道，pipeline中分为inbound事件和outbound事件，请求进来是inbound事件，当达到最后一个inbound事件处理器之后，会调用outbound事件处理器，然后沿着tail -> head顺序继续处理，所以，inbound和outbound的ChannelHandler添加顺序需满足以下条件：
* 多个inboundChannelHandler按照业务逻辑顺序依次进行addLast
* 多个outboundChannelHandler按照业务逻辑顺序的反序依次进行addLast
* 所有outboundChannelHandler必须在最后一个调用outbound方法的inboundChannelHandler之前

比如上例中http请求的处理流程是`decoder(解码) -> aggregator -> httpHandler(http业务处理器) -> encoder(编码)`，因为前面3个是inbound事件，最后的encoder(编码)是outbound事件，所以前面3个必须按照此顺序依次addLast，encoder只要在调用addLast(HttpHandler)之前执行addLast(encoder)即可。

> 小结

每个channel内部都会持有一个`ChannelPipeline`对象，pipeline默认实现`DefaultChannelPipeline`的内部维护了一个`DefaultChannelHandlerContext`链表。当channel对应的事件触发时，会回调ChannelPipeline对应的方法，比如channelActive、channelRead等。

channelPipeline的channelHandlerContext链表是“责任链”模式的体现（类似于tomcat中的filter链机制），一个请求的处理可能会涉及到多个channelHandler，比如decodeHandler、业务channelHandler和encodeHandler。

**参考资料：**

1、[https://segmentfault.com/a/1190000007309311](https://segmentfault.com/a/1190000007309311)


