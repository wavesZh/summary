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
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }

                while (nextSequence <= availableSequence)
                {
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






