---
layout: blog
title: netty入门实践
date: 2018-09-15 14:44:37
categories: [netty]
tags: [netty]
---

> netty是一个高性能、事件驱动的Java网络库，其内部隐藏了底层网络通信的细节，给开发者提供了易用灵活的API。netty一个主要的目的是使业务逻辑从网络基础设施应用程序中分离。不仅仅是Netty框架，其他框架的设计目的也大都是为了使业务程序和底层技术解耦，使程序员更加专注于业务逻辑实现，提高开发质量和效率。

<!--more-->

Netty为什么性能如此之高，主要在于依托系统层异步机制（IO复用/pipeline等），结合优秀的Reactor线程模型、事件处理机制，实现高性能的事件处理。

学习netty原理，可以按照如下步骤来学习：
* 了解内部主要的类及其功能
* 连接的处理流程（connect/read/write），NIOEventLoop机制
* pipeline机制
* 线程模型
* 内存管理
* 其他一些功能，包括但不限于拆包/粘包的处理、Nio epoll bug的处理、心跳机制

关于netty源码的构建，可以参考 [netty源码编译](https://github.com/ThinkInOpenSource/netty/issues/1)，笔者对fork了源码并添加了少许注释，感兴趣可以点击[netty](https://github.com/ThinkInOpenSource/netty)查看。

## netty核心类

* **Bootstrap和ServerBootstrap**：Netty应用程序通过设置bootstrap引导类来完成，该类提供了一个用于应用程序网络层配置的容器。Bootstrap服务端的是ServerBootstrap，客户端的是Bootstrap。
* **Channel**：Netty 中的接口 Channel 定义了与 socket 丰富交互的操作集：bind, close, config, connect, isActive, isOpen, isWritable, read, write 等等。
* **ChannelHandler**：ChannelHandler 支持很多协议，并且提供用于数据处理的容器，ChannelHandler由特定事件触发， 常用的一个接口是ChannelInboundHandler，该类型处理入站读数据（socket读事件）。
* **ChannelPipeline**：ChannelPipeline 提供了一个容器给 ChannelHandler 链并提供了一个API 用于管理沿着链入站和出站事件的流动。每个 Channel 都有自己的ChannelPipeline，当 Channel 创建时自动创建的。 
* **EventLoop**：EventLoop 用于处理 Channel 的 I/O 操作。一个单一的 EventLoop通常会处理多个 Channel事件。一个 EventLoopGroup 可以含有多于一个的 EventLoop 和 提供了一种迭代用于检索清单中的下一个。
* **ChannelFuture**：Netty 所有的 I/O 操作都是异步。因为一个操作可能无法立即返回，我们需要有一种方法在以后确定它的结果。出于这个目的，Netty 提供了接口 ChannelFuture,它的 addListener 方法注册了一个 ChannelFutureListener ，当操作完成时，可以被通知（ 不管成功与否） 。

下图说明了ChannelHandler和ChannelPipeline二者的关系：
{% asset_img 20180915043929817.png %}

Netty 是一个非阻塞、事件驱动的网络框架，Netty使用多线程（*一个EventLoop对应一个线程，一个EventLoopGroup中可能有多个EventLoop*）处理IO事件，这里的多线程处理IO事件不会涉及到事件的同步操作，因为一个Channel只会被添加到一个EventLoop中，后续该channel的事件都是由该EventLoop来响应的。

### buffer

ByteBuf是字节数据的容器，所有的网络通信都是基于底层的字节流传输，ByteBuf 是一个很好的经过优化的数据容器，我们可以将字节数据有效的添加到 ByteBuf 中或从 ByteBuf 中获取数据。为了便于操作，ByteBuf 提供了两个索引：一个用于读，一个用于写。我们可以按顺序读取数据，也可以通过调整读取数据的索引或者直接将读取位置索引作为参数传递给get方法来重复读取数据。

堆缓冲区ByteBuf将数据存储在 JVM 的堆空间，这是通过将数据存储在数组的实现。堆缓冲区可以快速分配，当不使用时也可以快速释放。它还提供了直接访问数组的方法，通过` ByteBuf.array()` 来获取 `byte[]`数据。 

堆缓冲区ByteBuf使用示例：
```java
ByteBuf heapBuf = ...;
if (heapBuf.hasArray()) {
    byte[] array = heapBuf.array();
    int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();
    int length = heapBuf.readableBytes();
    handleArray(array, offset, length);
}
```

直接缓冲区ByteBuf，在 JDK1.4 中被引入 NIO 的ByteBuffer 类允许 JVM 通过本地方法调用分配内存，其目的是通过免去中间交换的内存拷贝, 提升IO处理速度; 直接缓冲区的内容可以驻留在垃圾回收扫描的堆区以外。DirectBuffer 在 -XX:MaxDirectMemorySize=xxM大小限制下, 使用 Heap 之外的内存, GC对此”无能为力”，也就意味着规避了在高负载下频繁的GC过程对应用线程的中断影响。

## Netty使用示例

添加Maven依赖
```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.0.50.Final</version>
</dependency>
```

### server TCP版本
```java
public static void main(String[] args) {
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
 
    try {
        ServerBootstrap boot = new ServerBootstrap();
        boot.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .localAddress(8080)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new EchoHandler());
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
 
public class EchoHandler extends ChannelInboundHandlerAdapter {
 
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf in = (ByteBuf) msg;
 
        System.out.println(in.toString(CharsetUtil.UTF_8));
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
}
```

### server http版本
```java
public static void main(String[] args) {
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
 
    try {
        ServerBootstrap boot = new ServerBootstrap();
        boot.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .localAddress(8080)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline()
                            .addLast("decoder", new HttpRequestDecoder())
                            .addLast("encoder", new HttpResponseEncoder())
                            .addLast("aggregator", new HttpObjectAggregator(512 * 1024))
                            .addLast("handler", new HttpHandler());
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
 
public class HttpHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
 
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest msg) throws Exception {
        DefaultFullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1,
                HttpResponseStatus.OK,
                Unpooled.wrappedBuffer("hello netty".getBytes()));
 
        HttpHeaders heads = response.headers();
        heads.add(HttpHeaderNames.CONTENT_TYPE, HttpHeaderValues.TEXT_PLAIN + "; charset=UTF-8");
        heads.add(HttpHeaderNames.CONTENT_LENGTH, response.content().readableBytes()); // 3
        heads.add(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);
 
        ctx.writeAndFlush(response);
    }
}
```

### client tcp版本
```java
public static void main(String[] args) {
 
    EventLoopGroup group = new NioEventLoopGroup();
    try {
        Bootstrap b = new Bootstrap();
        b.group(group)
        .channel(NioSocketChannel.class)
        .option(ChannelOption.TCP_NODELAY, true)
        .handler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline p = ch.pipeline();
                //p.addLast(new LoggingHandler(LogLevel.INFO));
                p.addLast(new EchoClientHandler());
            }
        });
 
        // Start the client.
        ChannelFuture f = b.connect("localhost", 8081).sync();
        f.channel().closeFuture().sync();
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
        group.shutdownGracefully();
    }
}
 
public class EchoClientHandler extends ChannelInboundHandlerAdapter {
 
    private final ByteBuf message;
 
    public EchoClientHandler() {
        message = Unpooled.buffer(256);
        message.writeBytes("hello netty".getBytes(CharsetUtil.UTF_8));
    }
 
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ctx.writeAndFlush(message);
    }
 
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println(((ByteBuf) msg).toString(CharsetUtil.UTF_8));
        ctx.write(msg);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
 
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
 
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

> 小结

作为一个高性能异步网络库，netty的实现原理值得我们研究下，后续笔者会按照文章开头说的学习netty原理步骤顺序，分析netty的实现原理，文章难免有错误之处，欢迎指正交流~

