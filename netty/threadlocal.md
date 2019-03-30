# ThreadLocal vs FastThreadLocal 

ThreadLocal通过map获取value；通过index in array，效率更高 (注释上说的)

ThreadLocal的缺陷：内存泄露 (Memory Leak)

why？ 

```java
// ThreadLocal
public T get() {
    Thread t = Thread.currentThread();
    // 获取当前thread的属性 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        
        // 以当前ThreadLocal作为key查value
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```
1. ThreadLocalMap结构： key(WeakReference)=ThreadLocalMap, value=Object

2. 一般使用形式 ： `final static private ThreadLocal threadlocal = new ThreadLocal() { //init value }`

3. WeakReference 当被引用对象不可达时，下一次GC将被回收

4. 内存泄露的根本原因为无用对象占用的内存不能及时被回收释放

![](/images/threadlocal-referece.jpg)

几种内存泄露的场景

1. ThreadLocalMap中的key被回收了：key=null，但是value没有被回收。虽然get/set/remove等方法清除value的强引用，但是不是每次调用并且是不及时的
2. 与ThreadPool搭配使用，ThreadPool中的thread不会被回收直至JVM退出。虽然TheadLocalMap拥有对ThreadLocal对象的WeakReference，但是Thread拥有对
ThreadLocal对象的强引用。根据 "单挑引用链的可达性以最弱的一个引用类型来决定；多条引用链的可达性以最强的一个引用类型来决定"，不会回收。

怎么避免内存泄漏呢？

**每次使用完ThreadLocal，都调用它的remove()方法，清除数据**

FastThreadLocal没有以上第1种内存泄露的问题，但是还需要自己手动好remove












# link

* [使用ThreadLocal不当可能会导致内存泄露](http://ifeve.com/%E4%BD%BF%E7%94%A8threadlocal%E4%B8%8D%E5%BD%93%E5%8F%AF%E8%83%BD%E4%BC%9A%E5%AF%BC%E8%87%B4%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2/)
* [深入分析 ThreadLocal 内存泄漏问题](https://zhuanlan.zhihu.com/p/56214714)