# netty 源码


## client

最基本的echo开始

~~~java
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
             if (sslCtx != null) {
                 p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
             }
             p.addLast(new LoggingHandler(LogLevel.INFO));
             p.addLast(new EchoClientHandler());
         }
     });

    // Start the client.
    ChannelFuture f = b.connect(HOST, PORT).sync();

    // Wait until the connection is closed.
    f.channel().closeFuture().sync();
} finally {
    // Shut down the event loop to terminate all threads.
    group.shutdownGracefully();
}
~~~
### NioEventLoopGroup

[evenloop group better choice](https://github.com/netty/netty/wiki/Native-transports)

从初始化开始
~~~java
EventLoopGroup group = new NioEventLoopGroup();
~~~

层层递进，代码比较清晰。主要工作是初始化EventExecutor。

注意下MultithreadEventExecutorGroup
~~~java
children[i] = newChild(executor, args);
~~~

`newChild`是初始化EventExecutor的方法，在NioEventLoopGroup中实现

再额外提下MultithreadEventExecutorGroup中的EventExecutorChooser，用于选择负责处理连接的EventExecutor。

<!--
一个EventExecutor只绑定一个线程，一个连接只能由一个线程处理，一个线程可以处理多个连接。跟kafka有点类似，一个topic的一个partion只能由消费中的一个消费者线程消费，但是一个线程可以消费多个partion。就不存在什么业务并发问题了。有点想知道Kafka为什么有这么高的吞吐量。
-->

### Bootstrap

[options](https://docs.oracle.com/javase/7/docs/technotes/guides/net/socketOpt.html)

Bootstrap初始化，`group`绑定eventloop group；`channel`确定连接的类型（nio,oio），需与evenloop group配套使用； `option`socket配置；`handler` event handler， 通常使用`ChannelInitializer`注入多个handler。`initChannel`在channel注册到selector成功后调用。

~~~java
... ... 

.handler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch) throws Exception {
         ChannelPipeline p = ch.pipeline();
         p.addLast(new LoggingHandler(LogLevel.INFO));
         p.addLast(new EchoClientHandler());
     }
 });
~~~   

### connect

connect一路深入，Bootstrap.`doResolveAndConnect`是主要实现connect的方法。


#### initAndRegister

`initAndRegister`方法主要是生成channel并将其注册到selector，得到该channel与selector关联的selectionkey，绑定在channel上，并返回一个future。其中`init`方法将ChannelInitializer绑定到了channel中的pipline上，等channel注册成功再为channel添加handler。

注册成功后，才会进行真正的connect操作`doResolveAndConnect0`。如何知道注册成功呢？用一个listener在注册成功回调，调用该方法。


AbstractBootstrap.initAndRegister -> MultithreadEventLoopGroup.register -> SingleThreadEventLoop.register -> AbstractUnsafe.register。

1. 选择event executor与channel建立关联， 一个channel只能对应一个event executor。

这里就用到了EventExecutorChooser进行选择，有两种实现：其实都是轮询，一种是针对于event executor数量是2的指数的轮询，用位运算实现求余；一种就是非2指数的轮询，用正常求余的方式啦。可能2指数的方式更快点吧。

1.1 创建channel，这初始化的时候，生成了java nio中的socket channel。并为netty channel设置了`readInterestOp`。客户端为read，服务端为accept。为后面selector添加instest ops准备默认的receive data ops。

```java
final ChannelFuture initAndRegister() {
    ... ... 
    // 利用反射调用了之前bootstrap初始化设置的channel类型的构造函数。这里是NioSocketChannel。
    channel = channelFactory.newChannel();
    ... ...
}
```




[SelectionKey ops](https://docs.oracle.com/javase/7/docs/api/constant-values.html#java.nio.channels.SelectionKey.OP_ACCEPT)
[netty](https://www.imooc.com/article/31718)
[selector](http://tutorials.jenkov.com/java-nio/selectors.html)
[nio](http://www.importnew.com/26563.html)


~~~java
// SingleThreadEventLoop.register
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    // 将channel注册到selector
    promise.channel().unsafe().register(this, promise);
    return promise;
} 

// AbstractUnsafe.register
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    
    ... ...
    // 将channel与eventloop绑定
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
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}


~~~

`eventLoop.inEventLoop()`：当前线程是否是此eventLoop对应的线程。不是的话，则将task放入eventloop对应线程的task queue中，等待eventloop轮询执行。

SingleThreadEventExecutor.execute -> SingleThreadEventExecutor.startThread -> SingleThreadEventExecutor.doStartThread -> NioEventLoop.run

doStartThread方法调用会产生一个线程，负责处理处理内部task以及轮询selector获取感兴趣的事件（accept,connect,read,write等）。

该线程与eventloop建立关联：一个eventloop只有对应一个不变线程。

NioEventLoop.run

~~~java
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    // 为什么这里需要wakeup, 具体原因源码上已展示, 我的理解：
                    // 当1.Selector is waken up between 'wakenUp.set(false)' and 'selector.select(...)'. (BAD)
                    //  2. Selector is waken up between 'selector.select(...)' and 'if (wakenUp.get()) { ... }'. (OK)
                    // 设置wakeup为true太早了（可以通过addTask）。
                    // 第一种情况，通过查看wakeup方法的注释：If no selection operation is currently in progress then the next invocation of one of these methods will return
                    // immediately unless the {@link #selectNow()} method is invoked in the meantime. 
                    // 可以得知，接下来的selector.select(...)会被立即唤醒，并且'wakenUp.compareAndSet(false, true)'也会失败，导致正常想唤醒selector也会失败，selector.select(...)就堵塞了，直到wakeup=false。
                    // 解决方案： 提前wakeup，让下一次不堵啦。但是浪费：这两种情况都会进行wakeup。

                    // 感觉`selector.wakeup()`的使用好奇怪，当前未select调用该方法对下一次select生效，不好掌握其调用时间。
                    // ps: Invoking this method more than once between two successive selection operations has the same effect as invoking it just once. 
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            // -------01 start
            // 处理tasks和events的两种时间分配方式 
            // 主要看看`processSelectedKeys`
            if (ioRatio == 100) {
                try {
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
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
            // -------01 end
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
~~~

processSelectedKeys->processSelectedKeysOptimized->processSelectedKey

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop != this || eventLoop == null) {
            return;
        }
        // close the channel if the key is not valid anymore
        unsafe.close(unsafe.voidPromise());
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // 当处理完connect后，需要将connect op从interest ops中剔除，否则Selector.select(..)将一直返回connect
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            // 在finishConnect方法中将read op放入interest ops中，不细说，过
            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        // 这里提到了将readyOps设为0的原因：由于jdk bug，避免空转
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}

```


OP_WRITE事件主要是在发送缓冲区空间满的情况下使用。如：

~~~java
while (buffer.hasRemaining()) {
     int len = socketChannel.write(buffer);   
     if (len == 0) {
          selectionKey.interestOps(selectionKey.interestOps() | SelectionKey.OP_WRITE);
          selector.wakeup();
          break;
     }
}
~~~

因为写缓冲区在绝大部分时候都是有空闲空间的，所以如果你注册了写事件，这会



那主要先看看注册task`register0`方法。(先放放)

```java
// AbstractUnsafe.register0
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}

```


使得写事件一直处于就就绪，选择处理现场就会一直占用着CPU资源。所以，只有当你确实有数据要写时再注册写操作，并在写完以后马上取消注册。

~~~java
// OP_READ = 1 << 0
// OP_WRITE = 1 << 2
// OP_CONNECT = 1 << 3
// OP_ACCEPT = 1 << 4
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            // 0 是指socket-connect operations, 为什么要是0呢？socket-connect operations一般只有1，4，8
            // todo
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}

~~~



