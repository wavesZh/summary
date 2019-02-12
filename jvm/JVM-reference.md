## 1. 引言

JVM如何判断对象是否“死去”以便进行GC回收，目前有两种算法：引用计算算法（对象的引用数量），可达性分析算法（引用链是否可达）。这两种算法都与引用有关，那便要了解下引用是个什么东西。

## 2. 引用的类型


### 2.1 强引用（Strong Reference）

#### 2.1.1 概念
``` java 
Object object  = new Obejct();
String s = "hello";
```
强引用使得被引用对象不容易被GC: 只要强引用存在，垃圾收集器永远不会回收掉被引用的对象。 **即使在内存不足的情况下，JVM即使抛出OutOfMemoryError异常也不会回收这种对象** 。

### 2.2 软引用（Soft Reference）

### 2.2.1 概念
``` java
SoftReference<StringBuilder> softBuilder = new SoftReference<StringBuilder>(builder);
```
软引用用来描述一些还有用但是非必需的对象。**在系统即将发生内存溢出时，会将软引用关联的对象进行回收**。如果这次回收还没有足够的内存，才会抛出内存溢出异常。

### 2.2.2 使用场景

- `只有在内存不足的时候JVM才会回收该对象`这个特性很好解决了OOM的问题，并且也适合用于实现缓存。
- 引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。


### 2.3 弱引用（Weak Reference）

``` java
WeakReference<StringBuilder> weakBuilder = new WeakReference<StringBuilder>(builder);
```

弱引用也是描述非必须对象，但其强度比软引用更弱点。**被其引用的对象只能生存到下一次垃圾收集发生之前，无论当前内存是否足够**。


### 2.4 虚引用（Phanton Reference）

``` java
PhantomReference<StringBuilder> phantomBuilder = new PhantomReference<StringBuilder>(builder);
```

虚引用也称为幽灵引用或者幻影引用，是最弱的引用。一个对象是否有虚引用的存在不会影响其生存时间，也不能通过虚引用来获取一个对象的实例。使用虚引用的唯一目的就是被引用对象在被收集器回收时可以收到一个系统通知。


## 3 如何判断一个对象的可达性？

    单挑引用链的可达性以最弱的一个引用类型来决定； 
    多条引用链的可达性以最强的一个引用类型来决定；
    
```java
private void test_gc1(){
    //在heap中创建内容为"wohenhao"的对象，并建立a到该对象的强引用，此时该对象时强可及
    String a=new String("wohenhao");
    //对heap中的对象建立软引用，此时heap中的对象仍然是强可及
    SoftReference< ?> softReference=new SoftReference<String>(a);
    //对heap中的对象建立弱引用，此时heap中的对象仍然是强可及
    WeakReference< ?> weakReference=new WeakReference<String>(a);
    System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
    //heap中的对象从强可及到软可及
    a=null;
    System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
    softReference.clear();//heap中对象从软可及变成弱可及,此时调用System.gc()，
    System.gc();
    System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
}

private void test_gc2(){
//在heap中创建内容为"wohenhao"的对象，并建立a到该对象的强引用，此时该对象时强可及
    String a=new String("wohenhao");
    //对heap中的对象建立软引用，此时heap中的对象仍然是强可及
    SoftReference< ?> softReference=new SoftReference<String>(a);
    //对heap中的对象建立弱引用，此时heap中的对象仍然是强可及
    WeakReference< ?> weakReference=new WeakReference<String>(a);
    System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
    a=null;//heap中的对象从强可及到软可及
    System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
    System.gc();
    System.out.println("强引用："+a+"\n软引用"+softReference.get()+"\n弱引用"+weakReference.get()+"\n");
}
```


## link

1. [彻底理解JVM常考题之分级引用模型](https://mp.weixin.qq.com/s/gA7nZtmvgbNgdP5QipcYJQ)
2. [深入JVM对象引用](https://blog.csdn.net/dd864140130/article/details/49885811)

