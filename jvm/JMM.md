# JMM

Java内存模型杂记

## link

[Threads and Locks](https://docs.oracle.com/javase/specs/jvms/se6/html/Threads.doc.html)
[volatile](https://stackoverflow.com/questions/36853986/does-java-volatile-prevent-caching-or-enforce-write-through-caching)

## 基本概念

### 主内存和工作内存

在主内存中的变量被线程共享，作为线程间通线的桥梁。每个线程都有自己的工作内存（靠近cpu，使用的cache），线程所需的“公共变量“为主内存的变量的副本保存在工作内存。
不同线程之间不能直接访问对方的变量进行通信，需要通过做内存作为媒介。具体操作如下：

### JMM中8种原子性的操作

| opt | scope | detail |
| ---------- | :-----------:  | :-----------: |
|lock(锁定)| 主内存 | 将一个变量标示一个线程独占状态 |
|unlock(解锁) |主内存| 将锁定状态的变量释放，以便其他线程锁定|
|read(读取)|主内存|将主内存中变量传输到工作内存中，以便load|
|load(载入)|工作内存|将read传输的变量放入工作内存中的变量副本中|
|use(使用)|工作内存|将变量副本中的值传输给执行引擎|
|assign(赋值)|工作内存|将从引擎接收的值放入工作内存的变量副本中|
|store(存储)|工作内存|将变量副本传输到主内存中，以便wirte|
|write(写入)|主内存|将store传输的变量放入主内存中的变量中|


### 原子性 可见性 有序性


## volatile

对volatile变量的写操作编译后是一个lock前缀的指令，相当于一个内存屏障，使得别的cpu或者别的内核无效化其cache，使得volatile变量的修改对其他cpu可见。

volatile内存可见性前提：运算结果不依赖于该变量以及其他共享变量。 

ps: 对于valatile可见性，在网上看到了两种说法： 
1. 对volatile变量的更改进入main memory，使work memory的副本失效，导致接下来read 时会从main memory读取。
2. 每次read volatile变量直接从main memory读取。

有点晕！哪种是正确的？[这里](https://stackoverflow.com/questions/36853986/does-java-volatile-prevent-caching-or-enforce-write-through-caching)

<!--
例如： `volatile long i = 0 ;  i ++;` 对的，`i ++`依赖自己了。 那换成`i = i + 1`就可以了吗? 也许是可以了。

因为对于64位数据类型的操作，可以看作是2次32位的操作。这可能会导致多线程下，数据会出现异常：即非原值也非“半个变量”的数值。不过目前市面上的jvm的支持64位数据类型操作的原子性，故平时使用一般不需要将long,double声明为volatile。
-->

### 内存屏障

1. 保证屏障两边的指令操作不会因重排序颠倒顺序。
2. 写原值到主内存，刷新各个cpu的缓存。










