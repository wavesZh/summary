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
一个EventExecutor只绑定一个线程，一个连接只能由一个线程处理，一个线程可以处理多个连接。跟kafka有点类似，一个topic的一个partion只能由消费中的一个消费者线程消费，但是一个线程可以消费多个partion。就不存在什么并发问题了。有点想知道Kafka为什么有这么高的吞吐量。
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

`initAndRegister`方法主要是生成channel并将其注册到selector，得到该channel与selector关联的selectionkey，并返回一个future。其中`init`方法将ChannelInitializer绑定到了channel中的pipline上，等channel注册成功再为channel添加handler。

注册成功后，才会进行真正的connect操作`doResolveAndConnect0`。如何知道注册成功呢？用一个listener在注册成功回调，调用该方法。


AbstractBootstrap.initAndRegister -> MultithreadEventLoopGroup.register -> SingleThreadEventLoop.register -> AbstractUnsafe.register。

1. 选择event executor与channel建立关联， 一个channel只能对应一个event executor。

这里就用到了EventExecutorChooser进行选择，有两种实现：其实都是轮询，一种是针对于event executor数量是2的指数的轮询，用位运算实现求余；一种就是非2指数的轮询，用正常求余的方式啦。可能2指数的方式更快点吧。

1.1 创建channel，这初始化的时候，生成了java nio中的socket channel，并为客户端channel的selectionkey。

```java
final ChannelFuture initAndRegister() {
    ... ... 
    // 利用反射调用了之前bootstrap初始化设置的channel类型的构造函数。这里是NioSocketChannel.
    channel = channelFactory.newChannel();
    ... ...
}
```




[SelectionKey ops](https://docs.oracle.com/javase/7/docs/api/constant-values.html#java.nio.channels.SelectionKey.OP_ACCEPT)
[netty](https://www.imooc.com/article/31718)
[selector](http://tutorials.jenkov.com/java-nio/selectors.html)
[netty02](http://www.importnew.com/26563.html)


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

`eventLoop.inEventLoop()`：当前线程是否是此eventLoop对应的线程。不是的话，则将注册task放入eventloop对应线程的task queue中。

SingleThreadEventExecutor.execute -> SingleThreadEventExecutor.startThread -> SingleThreadEventExecutor.doStartThread -> NioEventLoop.run

doStartThread方法调用会产生一个线程，负责处理处理内部task以及轮询selector获取感兴趣的事件（accept,connect,read,write等）。

该线程与eventloop建立关联：一个eventloop只有对应一个不变线程。


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
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
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

因为写缓冲区在绝大部分时候都是有空闲空间的，所以如果你注册了写事件，这会使得写事件一直处于就就绪，选择处理现场就会一直占用着CPU资源。所以，只有当你确实有数据要写时再注册写操作，并在写完以后马上取消注册。



