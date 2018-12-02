# 源码解读

1. 创建disruptor

```java
public class Disruptor<T>
{
    private final RingBuffer<T> ringBuffer;
    private final Executor executor;
    // 消费者列表 evenHandler
    private final ConsumerRepository<T> consumerRepository = new ConsumerRepository<>();
    private final AtomicBoolean started = new AtomicBoolean(false);

    ......
    // 将得到的ringbuffer与disruptor关联起来
    public Disruptor(
    final EventFactory<T> eventFactory,
    final int ringBufferSize,
    final ThreadFactory threadFactory,
    final ProducerType producerType,
    final WaitStrategy waitStrategy)
	{
		this(
		    RingBuffer.create(producerType, eventFactory, ringBufferSize, waitStrategy),
		    new BasicExecutor(threadFactory));
	}
}


```
2. 创建ringbuffer.引出sequencer，sequencer与ringbuffer建立联系。 

sequencer根据`Sequence`所处的位置协调多线程下数据的读写。


```java
public static <E> RingBuffer<E> create(
        ProducerType producerType,
        EventFactory<E> factory,
        int bufferSize,
        WaitStrategy waitStrategy)
{
    switch (producerType)
    {
        case SINGLE:
            return createSingleProducer(factory, bufferSize, waitStrategy);
        case MULTI:
            return createMultiProducer(factory, bufferSize, waitStrategy);
        default:
            throw new IllegalStateException(producerType.toString());
    }
}	
```



3. 注册handler，将ringbuffer和handler组装成BatchEventProcessor.


```java

public final EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers)
{	// new Sequence[0]为barries。代表没有。
    return createEventProcessors(new Sequence[0], handlers);
}

EventHandlerGroup<T> createEventProcessors(
    final Sequence[] barrierSequences,
    final EventHandler<? super T>[] eventHandlers)
{	
	// 不运行在运行中注册handler
    checkNotStarted();

    final Sequence[] processorSequences = new Sequence[eventHandlers.length];
    final SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences);

    for (int i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++)
    {
        final EventHandler<? super T> eventHandler = eventHandlers[i];

        final BatchEventProcessor<T> batchEventProcessor =
            new BatchEventProcessor<>(ringBuffer, barrier, eventHandler);

        if (exceptionHandler != null)
        {
            batchEventProcessor.setExceptionHandler(exceptionHandler);
        }
        // 注册消费者中心，便于得到各个消费者的消费情况
        consumerRepository.add(batchEventProcessor, eventHandler, barrier);
        processorSequences[i] = batchEventProcessor.getSequence();
    }

    updateGatingSequencesForNextInChain(barrierSequences, processorSequences);

    // EventHandlerGroup 一般是用来指定handler执行顺序的
    return new EventHandlerGroup<>(this, consumerRepository, processorSequences);
}

private void updateGatingSequencesForNextInChain(final Sequence[] barrierSequences, final Sequence[] processorSequences)
{
    if (processorSequences.length > 0)
    {	
    	// 目前还看不懂gateing sequences是什么东西 插槽？？？
        ringBuffer.addGatingSequences(processorSequences);
        for (final Sequence barrierSequence : barrierSequences)
        {
            ringBuffer.removeGatingSequence(barrierSequence);
        }
        consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
    }
}
```


4. start disruptor。 

以BatchEventProcessor为例， 

```java
public RingBuffer<T> start()
{
    checkOnlyStartedOnce();
    for (final ConsumerInfo consumerInfo : consumerRepository)
    {
        consumerInfo.start(executor);
    }

    return ringBuffer;
}

public interface EventProcessor extends Runnable { ....... }
public final class BatchEventProcessor<T> {

	@Override
    public void run()
    {
    	......
        processEvents();
        ......
    }

	private void processEvents()
    {
        T event = null;
        long nextSequence = sequence.get() + 1L;

        while (true)
        {
            try
            {
                // 先等待barrier对应的handler处理完，得到下一个可处理序列/插槽
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null)
                {   
                    // 待处理序列预处理， 一般只是用来记录
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }

                while (nextSequence <= availableSequence)
                {public long waitFor(final long sequence)
        throws AlertException, InterruptedException, TimeoutException
    {
        checkAlert();

        long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
        // 存在barrier，并且得到 所有 barriers 中最小的消费序列，可以开始消费了
        if (availableSequence < sequence)
        {
            return availableSequence;
        }
        // single producer: 直接那得到
        // multi producer: 获取
        return sequencer.getHighestPublishedSequence(sequence, availableSequence);
    }
                    event = dataProvider.get(nextSequence);
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }

                sequence.set(availableSequence);
            }
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (running.get() != RUNNING)
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                exceptionHandler.handleEventException(ex, nextSequence, event);
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    }
}

```


5. ProcessingSequenceBarrier.waitFor(sequence),默认使用BlockingWaitStrategy  

```java

// sequence表示下一个消费序列
public long waitFor(final long sequence)
        throws AlertException, InterruptedException, TimeoutException
{
    checkAlert();

    long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
    // 不清楚什么情况 availableSequence < sequence， 可能更waitStrategy的种类有关，后面再研究
    if (availableSequence < sequence)
    {
        return availableSequence;
    }
    // 存在barrier，并且得到 所有 barriers 中最小的消费序列，可以开始消费了
    // single producer: 直接使用availableSequence
    // multi producer: 根据pushlishEvent时的规则获取有效的序列
    return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}

// BlockingWaitStrategy
public long waitFor(long sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException
{
    long availableSequence;
    // wait event publish
    // cursor表示已经写入事件的序列， 当低于下一个消费序列时，进行轮训等待，直至生产>=待消费
    if (cursorSequence.get() < sequence)
    {
        synchronized (mutex)
        {
            while (cursorSequence.get() < sequence)
            {
                barrier.checkAlert();
                // 当publish event唤醒
                mutex.wait();
            }
        }
    }
    // 如果不存在barrier，dependentSequence.get()将返回 Long.MAX_VALUE。 
    // 否则会轮询直到barrier已经消费完 >=sequence,当前handler才能继续处理
    while ((availableSequence = dependentSequence.get()) < sequence)
    {
        barrier.checkAlert();
        // 如果是java9,则会调用onSpinWait()进行空轮询，更加有效率，减少cpu的消耗
        ThreadHints.onSpinWait();
    }

    return availableSequence;
}
```

6. disruptor.publishEvent, 发布事件

```java
public void publishEvent(EventTranslator<E> translator)
{
    final long sequence = sequencer.next();
    translateAndPublish(translator, sequence);
}
```

7. sequencer分为single producer和multi producer

* MultiProducerSequencer

```java
@Override
public long next()
{
    return next(1);
}

@Override
public long next(int n)
{
    if (n < 1 || n > bufferSize)
    {
        throw new IllegalArgumentException("n must be > 0 and < bufferSize");
    }

    long current;
    long next;

    do
    {
        // 当前生产序列
        current = cursor.get();
        next = current + n;
        // 等到生产一圈后的生产数量 ？
        long wrapPoint = next - bufferSize;
        // gatingSequenceCache 初始值为 -1  barries最低消费序列
        long cachedGatingSequence = gatingSequenceCache.get();
        // 轮询直至生产和消费正常 
        // cachedGatingSequence > current 什么情况会消费快于生产？
        // 自己的理解，可能有误： 手动指定待消费序列 sequcencer.claim  
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)
        {
            // barriers最低消费序列
            long gatingSequence = Util.getMinimumSequence(gatingSequences, current);
            // 可以看作 next > gatingSequence + bufferSize, 表示produce快于消费。再produce就会覆盖尚未消费的序列
            if (wrapPoint > gatingSequence)
            {
                LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                continue;
            }

            gatingSequenceCache.set(gatingSequence);
        }
        else if (cursor.compareAndSet(current, next))
        {
            break;
        }
    }
    while (true);

    return next;
}
```

* SingleProducerSequencer

```java

public long next(int n)
{
    if (n < 1 || n > bufferSize)
    {
        throw new IllegalArgumentException("n must be > 0 and < bufferSize");
    }

    long nextValue = this.nextValue;

    long nextSequence = nextValue + n;
    long wrapPoint = nextSequence - bufferSize;
    long cachedGatingSequence = this.cachedValue;

    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
    {
        // valatile内存可见
        cursor.setVolatile(nextValue);  // StoreLoad fence

        long minSequence;
        while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))
        {
            LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin?
        }

        this.cachedValue = minSequence;
    }

    this.nextValue = nextSequence;

    return nextSequence;
}

```








