# ArraysList 

底层数据结构为数组. 
最好在初始化时指定 size，否则默认为10. 指定合适的 size, 不然频繁扩容（数组创建），影响性能.
扩容时机：数组不能再放入新元素; 扩容大小：1.5 * oldSize 或者 minSize

# Vector

线程安全。跟 ArrayList 类似。方法级别 sync. 扩容大小： 2 * oldSize 或者 oldSize + initialCapacity.

fail-fast. 使用 iterator 修改结构。for (:) 语法糖，实际还是使用 iterator，游标错乱


# CopyOnWriteArrayList

线程安全。使用 lock 和 volatile. 读多写少场景。 因为每次修改都必然要重新生成一次数组
	




