# java8-parallel 

1. fork-join
2. 与普通并发区别
3. 使用场景


## fork-join

分而治之思想。

### ForkJoinPool 
// ForkJoinPool<-AbstractExecutorService<-ExecutorService

1. fork: 开启一个新线程处理任务。  
2. join: 等待任务的处理线程处理结束并返回结果。  

### work stealing 工作窃取算法

[fork-join资料01](https://kaimingwan.com/post/java/forkjoinpooljie-du)
[fork-join资料02](https://www.jianshu.com/p/f777abb7b251)

多个worker thread分别处理各自的works (work queue), 如果其中有的thread提前处理完了， 就会从其thread的work queue的消费的另一端窃取work进行处理。

1. work queue是一个双向的，额外的开销。 
2. 充分利用线程资源并且减少了线程间竞争。 



代码就不解析了，看的我头晕......

主要分析下fork 和 join
```java



```

### paralle stream 

## 使用场景

