---
layout: blog
title: NioEventLoop机制
date: 2018-09-15 17:40:02
categories: [netty]
tags: [netty]
---

> NioEventLoop是netty中一个重要的组件，遵循Reactor模型，处理各种连接事件、任务（包括延时任务），一个NioEventLoop对应一个线程。

<!--more-->

## NioEventLoop

### bossLoopGroup和workerLoopGroup

对于服务端来说，bossLoopGroup负责接收client的连接请求，也就是调用accept，创建socket连接(Channel)的过程，之后会将该channel register到workerLoopGroup中的Selector中，后续该channel的读写事件都由workerLoopGroup来处理`（准确来说是由workerLoopGroup中某个NIOEventLoop来处理，workerLoopGroup中一般会包含多个NIOEventLoop，bossLoopGroup会轮询选择workerLoopGroup某个EventLoop，将channel注册到该EventLoop上去）`，关于bossLoopGroup和workerLoopGroup更多的细节，后续在netty线程模型中在分享。

### NioEventLoop

EventLoop是一个Reactor模型的事件处理器，一个EventLoop对应一个线程，其内部会维护一个selector和taskQueue，负责处理客户端请求和内部任务，内部任务如ServerSocketChannel注册和ServerSocket绑定操作等。

{% asset_img 20180915095944202.png %}

EventLoop处理请求事件，即selectionKey中ready的事件，如accept、connect、read、write等，由processSelectedKeys方法触发。处理完请求时间之后，会处理内部添加到taskQueue中的任务，如register0、bind0等任务，由runAllTasks方法触发。内部任务的添加一般由以下代码添加：
```java
if (eventLoop.inEventLoop()) {
    register0(promise);
} else {
    eventLoop.execute(new Runnable() {
        @Override
        public void run() {
            register0(promise);
        }
    });
}
```

### NioEventLoop#run

netty中`bossGroup/workerGroup`就是NioEventLoop的集合，NioEventLoop的主要流程逻辑代码见`NioEventLoop#run`，NioEventLoop处理各种连接事件、任务（包括延时任务）：
```java
protected void run() {
	for (;;) {
		/*
		 * 如果hasTasks，则调用selector.selectNow()，直接唤醒selector，
		 * 此时就算selector返回为0，那么也会执行switch后面的逻辑
		 */
		switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
			case SelectStrategy.CONTINUE: // -2
				continue;
			case SelectStrategy.SELECT: // -1
				select(wakenUp.getAndSet(false));
				if (wakenUp.get()) {
					selector.wakeup();
				}
				// fall through
			default:
		}

		cancelledKeys = 0;
		needsToSelectAgain = false;

		/*
		 * ioRatio调节连接事件和内部任务执行时间百分比，以IO事件执行时间为基准
		 */
		final int ioRatio = this.ioRatio;
		if (ioRatio == 100) {
			try {
				// 处理IO事件，包括connect/read/accept/write
				processSelectedKeys();
			} finally {
				// Ensure we always run tasks.
				runAllTasks();
			}
		} else {
			final long ioStartTime = System.nanoTime();
			try {
				processSelectedKeys();
			} finally {
				// 执行任务/延时任务
				final long ioTime = System.nanoTime() - ioStartTime;
				runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
			}
		}
	}
}
```

`NioEventLoop#run`的处理流程相关代码较多，这里为了方便理解原理，只展示处理流程图，有兴趣的小伙伴可以根据流程图自行阅读源码~
{% asset_img 20180915095829976.png %}

在处理accept事件时，默认轮训workerGroup中NIOEventLoop，将channel注册到到其中一个NIOEventLoop上。写事件触发时，一般是进行数据的写入动作或者数据flush操作，如下所示：
```java
if ((readyOps & SelectionKey.OP_WRITE) != 0) {
    // 写事件
    ch.unsafe().forceFlush();
}
```

关于任务处理，任务有普通任务（在taskQueue的任务）和延时任务（在scheduledTaskQueue的任务，scheduledTaskQueue底层数据结构是一个优先级队列 `PriorityQueue`），在执行`runAllTasks`方法时，会把scheduledTaskQueue中已经到达延迟执行时间的任务移到taskQueue中等待被执行。

> 小结

EventLoop是一个**Reactor模型的事件处理器**，一个EventLoop对应一个线程，其内部会维护一个selector和taskQueue，负责处理IO事件和内部任务。IO事件和内部任务执行事件百分比通过ioRatio来调节，ioRatio表示执行IO事件所占百分比。

EventLoop是一个通用的事件处理器，不管是监听事件还是IO读写事件，还是内部任务，都做了统一的处理。如果taskQueue为空时，默认调用`selector.select(timeoutMillis /*1000ms*/)`，也就是说最多等待1s返回，如果在这1s内添加了task，那么也是需要等待selector.select返回后才能被执行到的。这里其实就牵扯到一个task执行实时性的问题了，因为是在同一个线程做IO事件和内部task的执行，所以获取IO事件阻塞时间越短，那么对task的执行就越实时，但是如果IO事件阻塞事件很短，那么CPU空转的情况就比较严重，所以这里netty做了权衡，默认`selector.select`最多阻塞1s。

**参考资料**：

1、[https://www.jianshu.com/p/9acf36f7e025](https://www.jianshu.com/p/9acf36f7e025)




