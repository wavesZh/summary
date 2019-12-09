# Buffer

Buffer 实际就是一块内存空间，用于存储数据。JDK 和 Netty 各自实现了一套Buffer。

“为什么 Netty 不使用 JVM 自身的 Buffer, 而是新造轮子？” 带着这样的问题来思考探究下！ 

## JDK Buffer

JDK buffer 是指 `java.nio.Buffer`。其一般分为两类：Heap Buffer 和 Direct Buffer。

Heap Buffer 使用的 JVM 所管理的内存，底层结构是 byte[]。`ByteBuffer` 中其定义声明为 `final byte[] hb;`，一旦初始化，长度就固定了，不能像 ArrayList 一样支持动态扩容🙁。
Direct Buffer 使用堆外内存，不由 JVM 管理。在 Java 代码层面用地址表示，使用 `Unsafe` 进行内存空间的操作。

在  `java.nio.Buffer` 中存在几个重要的参数：position, limit, capacity。

Java Doc 描述如下： 

> A buffer's capacity is the number of elements it contains. The capacity of a buffer is never negative and never changes.   
A buffer's limit is the index of the first element that should not be read or written. A buffer's limit is never negative and is never greater than its capacity.   
A buffer's position is the index of the next element to be read or written. A buffer's position is never negative and is never greater than its limit.

> mark <= postition <= limit <= capacity

由于其读写都依赖 position, 所以读写切换必须调用 `flip()` 方法切换，这在开发中容易忘记🙁。

另外读写方法也分为两类：相对方法和绝对方法。相对方法的会影响 position 的位置，而绝对方法并不会。索引改变意味着 "覆水难收", 如果想用相对方法再对已经处理过的数据进行操作，这是不行，需要借助绝对方法或者改变索引位置的方法。

相对方法： `get()`, `put(byte b)`等
绝对方法： `get(int index)`, `put(int index, byte)`

可以看出，绝对方法是直接利用索引进行操作的。

## Netty Buffer

Netty Buffer 是指 `io.netty.buffer.ByteBuf`。其作用跟 JDK Buffer 一样，都是用与存储数据，但是其对于 JDK Buffer 的缺点🙁进行了优化。

例如 Heap Buffer 动态扩容，ByteBuf 底层也是依赖 byte[], 但是可以在写空间不足时进行扩容，最大为 Integer.MAX_VALUE。

另外JDK的读写切换需要调用 `flip()` 方法切换，而 ByteBuf 使用读写索引分别控制，规则如下：

```
+-------------------+------------------+------------------+
| discardable bytes |  readable bytes  |  writable bytes  |
|                   |     (CONTENT)    |                  |
+-------------------+------------------+------------------+
|                   |                  |                  |
0      <=      readerIndex   <=   writerIndex    <=    capacity
```

其 Buffer 类型分为三种： Heap, Direct, Composite。 Heap Buffer 重新实现了一套，没有依赖 ByteBuffer；Direct Buffer 底层还是依赖 `java.nio.ByteBuffer#allocateDirect`; 而 Composite Buffer 不是一种新的内存 Buffer，是复合 Buffer，类似一个容器，里面可以有多种类型的 buffer，其屏蔽了差异细节，提供了一个统一视图以便处理。

### Buffer 类型选择

如进行 IO 操作时使用 Direcy Buffer， 其余情况则使用 Heap Buffer，如编解码存储临时数据。而 Composite Buffer 的使用场景，参考《Netty In Action》中的 HTTP 消息。消息由 Header 和 Body 两个 Buffer 组成，如果使用 JDK 中的 ByteBuffer 进行消息发送，需要重新分配一个新的 Buffer 进行数据拷贝保存 Header 和 Body, 如果使用 Composite Buffer，则不需要。

为什么 IO 操作最好使用 Direct Buffer ？ [这里](https://www.zhihu.com/question/57374068/answer/152691891) 有答案，膜拜大神！由于IO 操作最终还是通过 Direct Buffer 进行的，这样就避免了额外数据拷贝，提高性能。













