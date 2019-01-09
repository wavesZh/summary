# HashMap

[Java 8系列之重新认识HashMap](https://zhuanlan.zhihu.com/p/21673805)


# ConcurrentHashMap


# WeakHashMap 

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






