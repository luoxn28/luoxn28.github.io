---
layout: blog
title: netty 特性杂谈
date: 2018-09-18 22:58:07
categories: [netty]
tags: [netty]
---

> netty作为业界最流程的NIO框架之一，除了稳定高性能的异步通信能力之外，还包含了其他一些有用的特性，比如对于TCP粘包/拆包的处理、对java nio epoll bug的处理、心跳机制...

<!--more-->

## TCP粘包/拆包处理

学习netty处理黏包和拆包，首先要知道什么是黏包和拆包问题？黏包和拆包的产生是由于**TCP拥塞控制算法**（比如angle算法）和**TCP缓冲区机制**导致的，angle算法简单来说就是通过一些规则来尽可能利用网络带宽，尽可能的发送足够大的数据。TCP（发送/接收）缓冲区会暂缓数据，并且是有最大容量的。

**黏包**的产生是由于一次TCP通信数据量较少，导致多个TCP数据合并在一起（这里的合并可能发生在发送缓冲区合并后发送，也可能发生在接收缓冲区合并后应用程序一次性读取）。**拆包**的产生是由于一次TCP通信数据量较大（比如超过了MTU），导致发送时分片发送，这样接收时是多次接收后才是一个完整的数据。

netty处理黏包和拆包问题，思路就是以定长方式读取接收到的数据来处理（*比如读取固定长度数据量、以TLV方式读取数据、以固定分隔符读取数据等*）。比如处理定长的消息解码器`FixedLengthFrameDecoder。`

## java nio epoll bug处理

epoll机制是Linux下一种高效的IO复用方式，相较于selector和poll机制来说。其高效的原因是将基于事件的fd放到内核中来完成，在内核中基于红黑树+链表数据结构来实现，链表存放发生事件的fd集合，然后在进行epoll_wait返回给应用程序，由应用程序来处理这些fd事件。

使用IO服用，Linux下一般默认就是epoll，Java NIO在Linux下默认也是epoll机制，但是JDK中epoll的实现却是有漏洞的，其中最有名的java nio epoll bug就是即使是关注的select轮询事件的key为0的话，NIO照样不断的从select本应该阻塞的`Selector.select()/Selector.select(timeout)`中wake up出来，导致CPU 100%问题。如下图所示：
{% asset_img bug.png %}

那么产生这个问题的原因是什么的？其实在 [https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6670302](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6670302) 上已经说明的很清楚了，比如下面是bug复现的一个场景：
```java
A DESCRIPTION OF THE PROBLEM :
The NIO selector wakes up infinitely in this situation..

0. server waits for connection
1. client connects and write message
2. server accepts and register OP_READ
3. server reads message and remove OP_READ from interest op set
4. client close the connection
5. server write message (without any reading.. surely OP_READ is not set)
6. server's select wakes up infinitely with return value 0
```

上面的场景描述的问题就是连接出现了RST，因为poll和epoll对于突然中断的连接socket会对返回的eventSet事件集合置为POLLHUP或者POLLERR，eventSet事件集合发生了变化，这就导致Selector会被唤醒，进而导致CPU 100%问题。根本原因就是JDK没有处理好这种情况，比如SelectionKey中就没定义有异常事件的类型。
```java
class SelectionKey {
    public static final int OP_READ = 1 << 0;
    public static final int OP_WRITE = 1 << 2;
    public static final int OP_CONNECT = 1 << 3;
    public static final int OP_ACCEPT = 1 << 4;
}
```

既然nio epoll bug存在，那么能不能规避呢？答案是有的，比如netty就很好的规避了这个问题，它的处理机制就是如果发生了这种情况，并且发生次数超过了SELECTOR_AUTO_REBUILD_THRESHOLD（默认512），则调用rebuildSelector()进行Selecttor重建，这样就不用管之前发生了异常情况的那个连接了。因为重建也是根据SelectionKey事件对应的连接来重新注册的，对应的代码在`NioEventLoop#select`方法。

## 心跳机制

netti可以支持心跳检测，通过`IdleStateHandler`来实现，心跳可以检测（连接的）远程端是否存活或者活跃。

netty的IdleStateHandler类，提供了3种类型的心跳检测，通过其构造方法可以看出：
```java
public IdleStateHandler(
        long readerIdleTime, long writerIdleTime, long allIdleTime,
        TimeUnit unit) {
    this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
}
```
* readerIdleTime：为读超时时间（服务端一定时间内未收到客户端消息）
* writerIdleTime：为写超时时间（服务端一定时间内未向客户端发消息）
* allIdleTime：所有类型的超时时间

netty的机制依托于IdleStateHandler类，从类名可看出，该类是一个channelHandler，是在initChannel时添加到channel的pipeline中的，netty心跳示例代码：
```java
public static void main(String[] args) {
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
 
    try {
        ServerBootstrap boot = new ServerBootstrap();
        boot.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .localAddress(60000)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline()
                                .addLast(new IdleStateHandler(5, 0,
                                        0, TimeUnit.SECONDS))
                                .addLast(new MyIdleStateHandler());
                    }
                });
 
        // start
        ChannelFuture future = boot.bind().sync();
        future.channel().closeFuture().sync();
 
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        // shutdown
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}
 
public class MyIdleStateHandler extends ChannelInboundHandlerAdapter {
 
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf in = (ByteBuf) msg;
 
        System.out.println("[" + Thread.currentThread().getName() + "] : recv: " + in.toString(CharsetUtil.UTF_8));
        ctx.write(msg);
    }
 
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
 
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
 
    /**
     * idle回调方法
     */
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            System.out.println(event.isFirst() + " " + event.state());
 
            ByteBuf result = ctx.channel().alloc().buffer();
            result.writeBytes("reader idle".getBytes(CharsetUtil.UTF_8));
            ctx.writeAndFlush(result);
        } else {
            ctx.fireUserEventTriggered(evt);
        }
    }
}
```

上面代码在在channel的pipeline中添加了一个IdleChannelHandler，入参为new IdleStateHandler(5, 0, 0, TimeUnit.SECONDS)表示在服务器端会每隔5秒来检查一下channelRead方法被调用的情况，如果在5秒内该链上的channelRead方法都没有被触发，就会调用userEventTriggered方法。

心跳机制是通过延时任务来做的，netty延时任务（比如ReaderIdleTimeoutTask）是由`scheduledTaskQueue`来管理的，每一个NioEventLoop中有一个scheduledTaskQueue，用来管理延时任务。scheduledTaskQueue的底层是一个DefaultPriorityQueue（优先级队列，基于数组的一个任务优先级队列，任务的优先级是由任务的到期时间决定，任务到期时间越近的越靠前，其实也就是在DefaultPriorityQueue的0号位置）。

## 关于netty的一些思考

* bossGroup将channel传递给childGroup中的NioEventLoop目前只有一种负载均衡策略，轮询策略，该策略在大部分情况下都是一个不错的负载均衡策略。这里还可以考虑引入其他负载均衡策略，或者是提供接口让开发者可自定义负载均衡策略。如果开发者可自定义负载均衡策略，childGroup可由程序自己控制，比如可以动态伸缩容childGroup的NIOEventLoop数量，这样可根据不同场景使用不同数量的NIOEventLoop。
* 其实netty也是一个**轻量级**的web容器（是不是想到了tomcat），netty内部是有任务队列和线程池的，可以对外提供接口，方便获取netty内部的运行状态信息，方便监控。
* netty可以将其部分模块组件化，应用系统可按需引用，比如我只想使用NIOEventLoop、只用时间轮（HashedWheelTimeout）等。

