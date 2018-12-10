# disrutor 为什么这么快？

## 1. 无锁，使用CAS，锁的效率比CAS低

CAS是也是一种锁，乐观锁。    
这里无锁是指多线程publish和get时不需要使用lock、synchronized等常见的悲观锁，可以对比`ArrayBlockingQueue`的poll和offer。
在publish里面有特殊的地方：wait strangy，等待策略大多数使用了`wait`和`notify`，但是触发场景是消费和生产速度不正常的情况。



```java
public long incrementAndGet()
{
    return addAndGet(1L);
}

public long addAndGet(final long increment)
{
    long currentValue;
    long newValue;

    do
    {
        currentValue = get();
        newValue = currentValue + increment;
    }
    while (!compareAndSet(currentValue, newValue));

    return newValue;
}


public boolean compareAndSet(final long expectedValue, final long newValue)
{
    return UNSAFE.compareAndSwapLong(this, VALUE_OFFSET, expectedValue, newValue);
}
```
CAS（compare and swap）-> Unsafe -> JNI（java nactive interface）。 



缺点：
* 轮训
* ABA
* 只支持对一个对象进行CAS


## 2. 避免了“伪共享”，避免了无关缓存行的刷新  	
[关于缓存行的介绍](https://blog.csdn.net/qq_27680317/article/details/78486220)

伪共享指的是多个线程同时读写同一个缓存行的不同变量时导致的 CPU 缓存失效。尽管这些变量之间没有任何关系，但由于在主内存中邻近，存在于同一个缓存行之中，它们的相互覆盖会导致频繁的缓存未命中，引发性能下降。

```java

abstract class RingBufferPad
{
    protected long p1, p2, p3, p4, p5, p6, p7;
}

// 专门放置ring buffer的需要的属性
abstract class RingBufferFields<E> extends RingBufferPad
{

    private static final int BUFFER_PAD;
    private static final long REF_ARRAY_BASE;
    private static final int REF_ELEMENT_SHIFT;

    ......
}
public final class RingBuffer<E> extends RingBufferFields<E> implements Cursored, EventSequencer<E>, EventSink<E>
{
	// 避开了Cache line伪共享
    public static final long INITIAL_CURSOR_VALUE = Sequence.INITIAL_VALUE;
    protected long p1, p2, p3, p4, p5, p6, p7;

    ......
}    
```
	JVM对象继承关系中父类属性和子类属性的内存地址是顺序排列的。

一般来说，cache line的大小为64byte（1byte = 8 bit），long的大小为8byte。 p1-p7占56byte。  
  
`RingBufferPad`中的 p1-p7作为前置缓存填充，`RingBuffer`中的p1-p7作为后置缓存填充，而`RingBufferFields`的中属性（rbf）大小肯定是超过8byte。故rbf肯定是单独占一个cache line，避免了“伪缓存”。 




	external: Linux 内核会将它最近访问过的文件页面缓存在内存中一段时间，这个文件缓存被称为 PageCache。


## 3. 循环队列，重复利用，减少了GC





