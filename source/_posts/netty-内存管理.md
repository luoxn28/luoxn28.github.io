---
layout: blog
title: netty 内存管理
date: 2018-09-16 19:24:10
categories: [netty]
tags: [netty]
---

> netty内存管理目的是提高内存申请/使用/回收效率，避免频繁gc，提高内存使用效率。

<!--more-->

## bytebuf

JDK为了解决网络通信中的数据缓冲问题，提供了ByteBuffer（heap或者直接内存缓存）来解决缓存问题，通过缓冲区来平衡网络io和CPU之间的速度差异，等待缓冲区积累到一定量的数据再统一交给CPU去处理，从而提升了CPU的资源利用率。

Netty 使用 reference-counting(引用计数)来判断何时可以释放 ByteBuf 或 ByteBufHolder 和其他相关资源，从而可以利用池和其他技巧来提高性能和降低内存的消耗。这一点上不需要开发人员做任何事情，但是在开发 Netty 应用程序时，尤其是使用 ByteBuf 和 ByteBufHolder时，你应该尽可能早地释放池资源。 Netty 缓冲 API 提供了几个优势：
* 可以自定义缓冲类型
* 通过一个内置的复合缓冲类型实现零拷贝
* 扩展性好，比如 StringBuilder
* 不需要调用 flip() 来切换读/写模式
* 读取和写入索引分开
* 方法链
* 引用计数
* Pooling(池)

### ByteBuf 字节数据容器

ByteBuf 类似于一个字节数组，最大的区别是读和写的索引可以用来控制对缓冲区数据的访问。下图显示了一个容量为16的空的 ByteBuf 的布局和状态，writerIndex 和 readerIndex 都在索引位置 0 ：
{% asset_img 20180916074710744.png %}

ByteBuf可以基于heap buffer，也可以基于direct buffer。使用direct buffer，通过免去中间交换的内存拷贝, 提升IO处理速度; 直接缓冲区的内容可以驻留在垃圾回收扫描的堆区以外。DirectBuffer 在 -`XX:MaxDirectMemorySize=xxM`大小限制下, 使用 Heap 之外的内存, GC对此”无能为力”,也就意味着规避了在高负载下频繁的GC过程对应用线程的中断影响。

注意：使用完ByteBuf之后，一定要release，否则会造成内存泄漏。区分ByteBuf底层是heap buffer还是direct buffer，可以根据ByteBuf.hasArray()来判断，因为heap buffer返回true（`heap上的ByteBuffy底层实现就是byte[] 数组`），direct buffer返回false。

### 复合缓冲区 COMPOSITE BUFFER

复合缓冲区是多个ByteBuf组合的视图，复合缓冲区就像一个列表，我们可以动态的添加和删除其中的 ByteBuf，JDK的 ByteBuffer 没有这样的功能。Netty 提供了 ByteBuf 的子类 CompositeByteBuf 类来处理复合缓冲区，CompositeByteBuf只是一个视图。注意：CompositeByteBuf.hasArray() 总是返回 false，因为它可能既包含堆缓冲区，也包含直接缓冲区。

例如，一条消息由 header 和 body 两部分组成，将 header 和 body 组装成一条消息发送出去，可能 body 相同，只是 header 不同，使用CompositeByteBuf 就不用每次都重新分配一个新的缓冲区。下图显示CompositeByteBuf 组成 header 和 body：
{% asset_img 20180916075100124.png %}

CompositeByteBuf使用示例：
```java
ByteBuf byteBuf1 = UnpooledByteBufAllocator.DEFAULT.buffer();
ByteBuf byteBuf2 = UnpooledByteBufAllocator.DEFAULT.heapBuffer();
 
byteBuf1.writeByte(1);
byteBuf2.writeByte(2);
 
CompositeByteBuf compositeByteBuf = Unpooled.compositeBuffer();
compositeByteBuf.addComponent(byteBuf1);
compositeByteBuf.addComponent(byteBuf2);
System.out.println(compositeByteBuf.getByte(0));
System.out.println(compositeByteBuf.getByte(1));
 
ByteBuf allByteBuf = Unpooled.wrappedBuffer(byteBuf1, byteBuf2);
System.out.println(allByteBuf.getByte(0));
System.out.println(allByteBuf.getByte(1));
```

### netty buffer

ByteBuf 是Netty中主要用来数据byte[]的封装类，主要分为Heap ByteBuf 和 Direct ByteBuf。为了减少内存的分配回收以及产生的内存碎片，Netty提供了PooledByteBufAllocator 用来分配可回收的ByteBuf，可以把PooledByteBufAllocator看做一个池子，需要的时候从里面获取ByteBuf，用完了放回去，以此提高性能。当然与之对应的还有 UnpooledByteBufAllocator，顾名思义Unpooled就是不会放到池子里，所以根据该分配器分配的ByteBuf，不需要放回池子有JVM自己GC回收。

在netty中，根据ChannelHandlerContext 和 Channel获取的Allocator默认都是Pooled，所以需要再合适的时机对其进行释放，避免造成内存泄漏。在传递过程中自己通过Channel或ChannelHandlerContext创建的但是没有传递下去的ByteBuf也要手动释放。为了帮助你诊断潜在的泄漏问题，netty提供了ResourceLeakDetector，该类会采样应用程序中%1的buffer分配，并进行跟踪，不过不用担心这个开销很小。
```java
// 第一种方式
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf in = (ByteBuf) msg;
 
    System.out.println(in.toString(CharsetUtil.UTF_8));
    // 调用ctx.write(msg)不必手动释放了，Netty会自行作释放操作，但是如果调用
    // ctx.write()两次或者调用ctx.write后又将该msg传递到了TailContext了，则就会报异常
    ctx.write(msg);
}
 
// 第二种方式
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf in = (ByteBuf) msg;
 
    System.out.println(in.toString(CharsetUtil.UTF_8));
 
    ByteBuf result = ctx.channel().alloc().buffer();
    result.writeBytes(in.toString(CharsetUtil.UTF_8).getBytes(CharsetUtil.UTF_8));
    ctx.write(result);
 
    // msg对应的ByteBuf释放工作交给TailContext来做
    ctx.fireChannelRead(msg);
}
 
// 第三种方式
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf in = (ByteBuf) msg;
 
    System.out.println(in.toString(CharsetUtil.UTF_8));
 
    ByteBuf result = ctx.channel().alloc().buffer();
    result.writeBytes(in.toString(CharsetUtil.UTF_8).getBytes(CharsetUtil.UTF_8));
    ctx.write(result);
 
    // 手工释放ByteBuf
    in.release();
}

// 第四种方式
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf in = (ByteBuf) msg;

    ctx.write(msg);
    // 增加一次ref计数
    ((ByteBuf) msg).retain();
    ctx.write(msg);
}
```

参考资料：
1、[https://my.oschina.net/ywbrj042/blog/902321](https://my.oschina.net/ywbrj042/blog/902321)
2、《Netty实战》
3、[https://www.cnblogs.com/xys1228/p/6088805.html](https://www.cnblogs.com/xys1228/p/6088805.html)
4、[https://blog.csdn.net/u012807459/article/details/77259869](https://blog.csdn.net/u012807459/article/details/77259869)
5、[
Netty学习之旅------源码分析Netty内存池分配机制初探--PoolArena、PoolChunk、PoolSubpage等数据结构分析](https://blog.csdn.net/prestigeding/article/details/54598967)
