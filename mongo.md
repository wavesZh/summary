# mongo

## link

[Mysql 索引原理及优化](http://www.cnblogs.com/hellojesson/p/6001685.html)

## index 索引

使用B-Tree数据结构。 // todo index和data是否分离存储

### 索引类型

1. 单属性

### 如何选择合适索引

合适的index有助于提高query效率。mongo中的index有些奇怪，主要是根据属性的值的顺序（1，-1）来建立索引。

好的index能让查询scan尽可能少的document。用explain来解析。一般来说，如果查询scan到的row超过总row的一半，其实就没有必要利用索引啦。使用索引也是会耗费一定的存储空间（RAM以及ROM）。

index的建立要依赖于数据的实际使用情况。

但是要注意：

1. 当field的值区分度低时，不适合index，需要和其他field组成复合index。
2. 要知道哪些操作是不用到index的：$ne,$nin。
3. 不要建立过多index，会影响到写的性能。
3. index的顺序。

多个单键索引 vs 一个复合索引

查询如果涉及多个单键索引，mongo会选择包含row少的索引scan，然后剩下的索引fetch

## shard 分片

shard key 不能进行更新，`Error:
Error when saving document: 1 After applying the update to the document {createTime: new Date(1542886184859) , ...}, the (immutable) field 'createTime' was found to have been altered to createTime: new Date(1542886181859)`。
