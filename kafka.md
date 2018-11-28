# Kafka 杂记

日常kafka使用零碎所得。

## 基本概念

* consumer group下可以有一个或多个consumer instance，consumer instance可以是一个进程，也可以是一个线程
* group.id是一个字符串，唯一标识一个consumer group
* consumer group下订阅的topic下的每个分区只能分配给某个group下的一个consumer(当然该分区还可以被分配给其他group)

## 杂记

1. spring应用启动时，consumer通过KafkaListenerAnnotationBeanPostProcessor.postProcessAfterInitialization方法进行初始化, 根据`containerFactory = "kafkaListenerContainerFactory"`从spring容器中找出匹配的bean。 
由KafkaConsumerConfig可知kafkaListenerContainerFactory的作用域是single，单例。 
如果多个@KafkaListener使用同一个kafkaListenerContainerFactory，即同一个消费组订阅多个topic，每个topic的listener数量会低于预期值concurrency。

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