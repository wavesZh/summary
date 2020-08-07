# Kafka 杂记

日常kafka使用零碎所得。

## link

https://www.cnblogs.com/huxi2b/p/6223228.html

## 基本概念

* consumer group下可以有一个或多个consumer instance，consumer instance可以是一个进程，也可以是一个线程
* group.id是一个字符串，唯一标识一个consumer group
* consumer group下订阅的topic下的每个分区只能分配给某个group下的一个consumer(当然该分区还可以被分配给其他group)

## 杂记

1. spring应用启动时，consumer通过KafkaListenerAnnotationBeanPostProcessor.postProcessAfterInitialization方法进行初始化, 根据`containerFactory = "kafkaListenerContainerFactory"`从spring容器中找出匹配的bean。由KafkaConsumerConfig可知kafkaListenerContainerFactory的作用域是single。  
如果多个@KafkaListener使用同一个kafkaListenerContainerFactory，即同一个消费组订阅多个topic，每个topic的listener数量会低于预期值concurrency？

```java
@Component
public class consumer01 {
	@KafkaListener(topics = "topic01", containerFactory = "kafkaListenerContainerFactory")	
	public void listen(ConsumerRecord<Long, String> record) {
		// do something
	}
}

@Configuration
@EnableKafka
public class KafkaConsumerConfig {

	@Bean
//	@Scope(scopeName = "prototype")
	public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Integer, String>> kafkaListenerContainerFactory() {
		ConcurrentKafkaListenerContainerFactory<Integer, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
		factory.setConsumerFactory(consumerFactory());
		factory.setConcurrency(concurrency);
		factory.getContainerProperties().setPollTimeout(30000);
		return factory;
	}

	... ...
}

```
通过实验，从VisualVM可以看到各个listener数量达到预期。 
故查看listener的启动源码：

从KafkaListenerAnnotationBeanPostProcessor.postProcessAfterInitialization开始。

```java

//ConcurrentMessageListenerContainer.doStart
protected void doStart() {
	if (!isRunning()) {
		... ...
		// 在创建listenerCotainer时进行concurrency赋值(KafkaListenerEndpointRegistry.registerListenerContainer)
		for (int i = 0; i < this.concurrency; i++) {
			KafkaMessageListenerContainer<K, V> container;
			if (topicPartitions == null) {
				container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties);
			}
			else {
				container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties,
						partitionSubset(containerProperties, i));
			}
			if (getBeanName() != null) {
				container.setBeanName(getBeanName() + "-" + i);
			}
			if (getApplicationEventPublisher() != null) {
				container.setApplicationEventPublisher(getApplicationEventPublisher());
			}
			container.setClientIdSuffix("-" + i);
			// 调用KafkaMessageListenerContainer.start().
			container.start();
			this.containers.add(container);
		}
	}
}



// KafkaMessageListenerContainer.doStart
// 生成message listener
protected void doStart() {
	... ... 
	// 获取在createListenerContainer中生成
	ContainerProperties containerProperties = getContainerProperties();
	... ...
	// 生成listener线程的executor在此创建，与concurrency不相关。
	if (containerProperties.getConsumerTaskExecutor() == null) {
		SimpleAsyncTaskExecutor consumerExecutor = new SimpleAsyncTaskExecutor(
				(getBeanName() == null ? "" : getBeanName()) + "-C-");
		containerProperties.setConsumerTaskExecutor(consumerExecutor);
	}
	... ...

	this.listenerConsumerFuture = containerProperties
				.getConsumerTaskExecutor()
				.submitListenable(this.listenerConsumer);
}
```


2. 埋坑： max.poll.interval.ms 

kafka版本： 0.10.1

[Difference between session.timeout.ms and max.poll.interval.ms](https://stackoverflow.com/questions/39730126/difference-between-session-timeout-ms-and-max-poll-interval-ms-for-kafka-0-10-0)。
[困扰许久的Kafka Rebalance问题](https://zhuanlan.zhihu.com/p/46963810)  
good:[Kafka client 消息接收的三种模式](https://blog.csdn.net/laojiaqi/article/details/79034798)

当consumer的max.poll.interval.ms比业务处理时间要短的话，会导致rebalance，offset提交失败，进而重复消费message。


复盘： 发现重复消费的情况，首先想到offset没有提交成功->放开debug日志，查看spring kafka的具体运行日志->无果，可能是auto.commit=true造成的，改为false，用spring kafka自己的提交方式->无效，将业务代码注释，有效，故认为业务代码处理时间过长造成的->业务时长不能缩短，那就尝试更改consumer配置->首先将session.time.out调大，无效（后面发现这个只会影响heartbreat线程（coordinator和conumser group一对一），确保存活，不影响consumer线程）-> 后面看到日志提示， `You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.record™s. `增加拉取频率（max.poll.interval.ms）降低拉取批次数量（max.poll.records），有效


ps: 在 spring-kafka 中, enable.auto.commit=false, 并不是指禁止自动提交，而是采用 kafka 默认的提交方式; true 则表示采用 spring 人工提交方式。false, 当下一次poll时提交之前poll下来的offset；enable.auto.commit=true，后台线程定期提交（consumerCoordinator.maybeAutoCommitOffsetAsync）。如果想要实现手动提交，除了 enable.auto.commit=false, 还需要指定 AckMode=MANUAL


HeartbeatThread 负责监听节点的存活情况。

code:

```java
// 如果当前时间-最近一次poll时间>maxPollInterval,则判定该节点可能死亡，需将从group中剔除，导致rebalance。
// 
if (heartbeat.pollTimeoutExpired(now)) {
	maybeLeaveGroup();
}
public boolean pollTimeoutExpired(long now) {
    return now - lastPoll > maxPollInterval;
}
```

rebalance导致提交失败: ConsumerCoordinator.sendOffsetCommitRequest

~~~java
final Generation generation;
if (subscriptions.partitionsAutoAssigned())
	// 客户端状态不再是STABLE
    generation = generation();
else
    generation = Generation.NO_GENERATION;

// if the generation is null, we are not part of an active group (and we expect to be).
// the only thing we can do is fail the commit and let the user rejoin the group in poll()
if (generation == null)
    return RequestFuture.failure(new CommitFailedException());
~~~

`ConsumerRebalanceListener`了解下，消费者rebalance的回调。可以用来保存rebalance时的offset以及提交等,这个可以精准消费，避免rebalance后的重复消费。

3. 误区 auto.offset.reset：latest and earliest

link: [Why is Kafka consumer ignoring my “earliest” directive in the auto.offset.reset parameter and thus not reading my topic from the absolute first event?
](https://stackoverflow.com/questions/49945450/why-is-kafka-consumer-ignoring-my-earliest-directive-in-the-auto-offset-reset)

auto.offset.reset都是在consumer起初开始消费，没有offset记录时起作用。earliest不能重复消费消息，只是初始化消费时从最早的数据开始消费，同理latest只是在初始化消费时，消费最新的数据。要想重新消费旧数据，可以通过更改group name和`seekToBeginning()`实现。



4. kafka异步发送消息

kafka默认就是异步`Future future = producer.send(record)`, 同步就话，`producer.send(record).get()`。根据业务自行选择。

发送结果可以通过future来获取，但是会阻塞。也可以通过自定义`ProducerListener`中的来异步处理结果。由`KafkaTemplate.doSend`可知。


5. 手动提交模式但不提交，会重复消费吗？

不会重复消费， 但是 reblanace 后，会重复消费。


6. kafka 异步消费

kafka 异步消费

a. offset 管理

批量拉取后的数据都由消费业务端处理，认为数据已消费，可以提交 offset。后续处理失败的补偿由业务方处理，不交由kafka

b. 限流机制，如果业务消费过慢，而consumer 不断拉取，消息将堆积在内存中。

设置阀值，如果超出阀值，则阻塞不进行拉取，阻塞时间不能过长导致 Rebalance。
如果阻塞后，pending 任务数仍超出阀值：
	1. 继续阻塞 -> rebalance -> 数据重复消费 不好
	2. 继续拉取 -> 内存数据堆积 -> 总会爆掉

选择方案2，消费能力不足，增加线程数量

c. 优雅停机策略

停止数据拉取->等待全部任务完成，


7. 批量消费

批量消费的关键不是拉取动作，而是提交偏移量这个动作。拉取数据一般来说都是批量拉取，而不会只拉取单条数据。那批量消费的优势在哪呢？体现在提交动作上。如果逐条提交，频繁调用api触发网络操作，性能不好.



