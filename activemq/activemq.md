# activemq 杂记

日常学习杂记

## link

[探究SpringJMS+ActiveMQ消息阻塞之谜](https://www.jianshu.com/p/0cf602f96e25)


## 杂记

### spring activemq consumer中的一些配置

#### taskExecutor and (concurrentConsumers | maxConcurrentConsumers)

taskExecutor 可以理解为是一个业务处理的thread pool, 会影响到消费速度。默认的taskExecutor是`SimpleAsyncTaskExecutor`且没有线程数量限制（意外发现spring kafka默认也是使用这个）。当然也可以自定义。

concurrentConsumers | maxConcurrentConsumers 表示最低 | 最高消费者数量，这些线程负责将mq中的消息丢给taskExecutor去处理。如果maxConcurrentConsumers没有设置，maxConcurrentConsumers将默认与concurrentConsumers一致。

日常如果消息发生堆积，那么可以调高些concurrentConsumers | maxConcurrentConsumers。



### activemq admin console

link: [Difference between Pending Messages and Enqueue Counter in Active MQ?](https://stackoverflow.com/questions/7786086/difference-between-pending-messages-and-enqueue-counter-in-active-mq)

#### pending messages、enqueued messages and dequeued messages

* pending messages: 待消费的消息，即堆积的消息量。准确来说是待放入队列消息的数量。
* enqueued messages: 进入队列的消息数量。重启后会清零
* dequeued messages: 出去队列的消息数量。重启后会清零

### purge and delete

purge: 

delete: 删除队列

### 消息类型

|消息类型|说明|
|:-----|:-----|
|StreamMessage|一种主体中包含Java基元值流的消息。其填充和读取均按顺序进行|
|MapMessage  |一种主体中包含一组键--值对的消息。没有定义条目顺序|
|TextMessage|一种主体中包含Java字符串的消息（例如，XML消息）|
|ObjectMessage|一种主体中包含序列化Java对象的消息|
|BytesMessage|一种主体中包含连续字节流的消息|









