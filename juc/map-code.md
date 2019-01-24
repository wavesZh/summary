# map

put and get 

## HashMap

### 死循环现象

hashmap中的死循环主要是因为解决hash冲突的链表产生了回环，查询死循环。


jdk7存在这种现象，jdk8 hashmap做了很多优化，不会再发生了。


回环产生在扩容阶段，重hash没有做同步操作。

#### jdk7 

主要代码如下： 

~~~java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            // 可以看出其重hash是倒序的。
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
~~~

产生回环的关键代码：
```java
e.next = newTable[i];
newTable[i] = e;
```
由上可知，其向head插入重hash的节点。

回环场景：
线程1，线程2同时对对A节点进行重hash，线程1已经完成节点的插入，newTable[i] = A, 此时线程2才刚刚开始，导致newTable[i] -> A = A，回环。如果是中间节点不可怕，后面会重新next，但是会丢失部分数据。如果对最后一个节点进行重hash发生这种情况，则over，查询该节点死循环。


#### jdk8

resize主要发生在table初始化或者2次幂扩容。

主要代码如下：

~~~java
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
Node<K,V> next;
do {
    next = e.next;
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
    else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
~~~

> ps: index = (n - 1) & hash, n为table容量。

jdk8，index不需要重新再计算了，只需要关注index值重计算后的新增位为0还是1。由`e.hash & oldCap`可得。

jdk8相对于jdk7的插入方式不同，其是向tail插入节点，这样就算中间节点发生上面的并发场景，也不怕，也不会丢失数据。尾节点产生回环，最后有`loTail.next = null`来去除回环。

### link

[Java 8系列之重新认识HashMap](https://zhuanlan.zhihu.com/p/21673805)
[Why does HashMap.put both compare hashes and test equality?](https://stackoverflow.com/questions/36100482/why-does-hashmap-put-both-compare-hashes-and-test-equality)

## ConcurrentHashMap

同步方式：cas + synchronized

jdk8之前是用分段锁的方式控制并发，但是效率稍低。jdk8将锁的粒度优化到每一个节点。

table.length >= MIN_TREEIFY_CAPACITY && binCount >= TREEIFY_THRESHOLD -> treeifyBin

put: 

1. helpTransfer
2. tryPresize




## WeakHashMap 

特性： 结构和HashMap基本一致，主要区别体现在Entry上

```java
 private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {

 	Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue);
      	... ...
    }
 }

 public class WeakReference<T> extends Reference<T> {

    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}

```

Entry继承了WeakReference，故而其有了WeakReference的特性：当被引用对象不可达时，下一次GC将被回收。 
有上可知，key为被引用的对象。key不可达时被GC回收。那values什么时候回收呢？ 

现看看get(key)中的getTable()方法:


```java
private Entry<K,V>[] getTable() {
    expungeStaleEntries();
    return table;
}
// 从表中清除过时的元素
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

里面的expungeStaleEntries()方法正是帮助GC回收的地方，该方法有3个地方调用：getTable(), size(), resize(newCapacity)

通过将value置null，帮助其GC。

```java
public static void test_weakhashmap() {

    WeakHashMap<String, User> map = new WeakHashMap<>(16);

    String id_1 = new String("1");
    String id_2 = new String("2");
    User user_1 = new User("1", "tao");
    User user_2 = new User("2", "hui");

    map.put(id_1, user_1);
    map.put(id_2, user_2);

    SoftReference<String> r = new SoftReference<>(id_1);
    id_1 = null;

    System.gc();

    System.out.println(map);

    r = null;

    System.gc();

    System.out.println(map);
}

```






