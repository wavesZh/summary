# JVM

## JVM运行时数据区

### 1. 程序计数器

程序计数器（program counter register）是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。通过改变其值以选取下一条需要执行的指令。

JVM的多线程是通过线程轮流切换并分配处理器时间片来实现的。为了当线程切换回来时，能恢复到正确的执行位置，所以每条线程都有独立的程序计数器，且互不干扰。由此可见，程序计数器所属内存属于“线程私有”内存。

### 2. java虚拟机栈

java虚拟机栈（java virtual machine stacks）也是线程私有，生命周期与线程相同。虚拟机栈描述的是java方法执行的内存模型：每个方法执行的时候都会创建一个栈帧用于存储
*`局部变量表、操作数栈、动态链接、方法出口`* 等信息。  

局部变量表存放了 `编译期可知`  的各种基本数据类型、对象引用（reference类型，它不等同与对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。  

在JVM规范中，对虚拟机栈区域规定了2种异常类型：
1. StackOverflowError。当线程请求的栈深度（栈帧的数量）大于虚拟机所允许的深度时抛出。
2. OutOfMemoryError。当虚拟机栈扩展时无法申请到足够的内存时抛出。

### 3. 本地方法栈

本地方法栈（native method stack）与虚拟机栈作用类似，区别在于虚拟机栈为虚拟机执行的java方法服务，而本地方法栈则是为Native方法服务。

### 4. Java堆

Java堆（java heap）的唯一目的是存放对象实例。其是垃圾收集器主要管理的区域，因此也叫做GC堆（garbage collected heap）。其内存空间在虚拟机启动是创建，大小通过`-Xmx`和`-Xms`设置。

目前GC采取的策略基本是分代收集，所以Java堆还可以分为：`新生带`和`老年代`:再细致有`Eden空间、From Survivor空间、To Survivor空间`等。

在JVM规范中，对虚拟机栈区域规定： 
如果在堆中没有内存完成实例分配，并且堆也无法扩展时，会抛出OutOfMemoryError。

>tips: Java堆默认为物理内存的1/4但小于1G，默认当空余堆内存小于40%时，JVM会增大Heap到-Xmx指定的大小，可通过-XX:MinHeapFreeRation=来指定这个比列；当空余堆内存大于70%时，JVM会减小heap的大小到-Xms指定的大小，可通过XX:MaxHeapFreeRation=来指定这个比列，对于运行系统，为避免在运行时频繁调整Heap的大小，通常-Xms与-Xmx的值设成一样。

### 5. 方法区

方法区与 Java 堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做 Non-Heap（非堆），目的应该是与 Java 堆区分开来。

经常将方法区叫做“永久代（PermGen）”，但是两者有本质区别：方法区是JVM的规范，而永久代是JVM规范的一种实现，并且只有HotSpot有永久代。

JDK8，以元空间（Metaspace）取代了永久代。元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。JDK7之前(包括JDK7)拥有"永久代"(PermGen space),用来实现方法区。但在JDK7中已经逐渐在实现中把永久代中把很多东西移了出来，比如:符号引用(Symbols)转移到了native heap,运行时常量池(interned strings)转移到了java heap；类的静态变量(class statics)转移到了java heap.

<img src="img/heap.jpeg"/>	

ps: 动态代理可能会导致元空间oom。

## 三、参考资料
* [Java虚拟机的内存组成以及堆内存介绍](http://www.hollischuang.com/archives/80)
* [深入了解Java虚拟机]()
* [Java8内存模型—永久代(PermGen)和元空间(Metaspace)](http://www.cnblogs.com/paddix/p/5309550.html)
* [JVM系列之实战内存溢出异常](https://juejin.im/post/5a162f16f265da432c23839a)
