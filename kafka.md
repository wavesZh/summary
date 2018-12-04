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

[Difference between session.timeout.ms and max.poll.interval.ms](https://stackoverflow.com/questions/39730126/difference-between-session-timeout-ms-and-max-poll-interval-ms-for-kafka-0-10-0)

当consumer的max.poll.interval.ms比业务处理时间要长的话，会导致rebalance，offset提交失败，进而重复消费message。


复盘： 发现重复消费的情况，首先想到offset没有提交成功->放开debug日志，查看spring kafka的具体运行日志->无果，可能是auto.commit=true造成的，改为false，用spring kafka自己的提交方式->无效，将业务代码注释，有效，故认为业务代码处理时间过长造成的->业务时长不能缩短，那就尝试更改consumer配置->首先将session.time.out调大，无效（后面发现这个只会影响heartbreat线程（coordinator和conumser group一对一），取保存活，不影响consumer线程）-> 后面看到日志提示， You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.records. 增加拉取频率（max.poll.interval.ms）降低拉取批次数量（max.poll.records），有效


ps: auto.commit=true, 当下一次poll时提交之前poll下来的offset；auto.commit=false，当

具体原理不是很清晰了解

code:

```java

```







