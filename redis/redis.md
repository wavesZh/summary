# Redis


## 持久化：AOF与RDB

### AOF(Append Only File)

过程持久化：写操作持久化

追加写操作至文件末尾

AOF 方式：`appendonly yes`

文件同步：

1. 写操作
2. 将写内容放入缓冲区
3. （系统/手动）刷盘

同步策略：

* always。每个写命令立即同步。受硬盘性能限制且损伤硬盘
* everysec。每秒同步。当硬盘忙于执行写操作，Redis 会自适应达到最佳写入速度
* no。操作系统决定何时同步


AOF 文件大小过大，磁盘空间占用过大以及 Redis 初始化时间过长
相应处理方法： 压缩合并（rewrite）

rewrite 方式： 

* `BGREWRITE` fork 子进程异步 rewrite
* 自动执行 `auto-aof-rewrite-percentage`(频率) `auto-aof-rewrite-min-size`（文件大小）


### RDB(Snapshotting | Redis Database) 

**快照**，某个时间点的 Redis 的数据。结果持久化。

快照持久化适用于允许部分数据丢失的场景。当系统发生崩溃，会丢失最近一次生成快照之后更改的数据

快照创建方式：

* 手动发送命令（`BGSAVE`/`SAVE`）
* 配置 `save time size` 配置选项
* 关闭/终止服务（正常关闭前肯定是需要持久化，以便下次启动初始化数据）
* 主从服务器连接 `SYNC`

快照流程：

Redis 主进程 fork 出一个子进程将数据集进行快照并替换旧快照

fork 操作会阻塞 Redis。数据集越大，花费的时间越长


## 复制

主从架构。高可用以及负载。

设置从服务器： `slaveof host port`

主从连接时会进行初始化：主快照并发送给从

主从链：主->主从->从，例如树形结构。从服务器过多会导致主服务器忙于I/O，复制过大，故从服务器可以拥有自己的从服务器

AOF（系统崩溃） + 复制（硬盘损坏），增强 Redis 对于系统崩溃的抵抗能力

## 系统故障处理

验证持久化文件： AOF(`redis-check-aof`) RDB9(`redis-check-dump`)

故障主服务器更换：

1. `SAVE` 生成快照文件
2. 快照文件发送给新服务器
3. 以快照启动 redis
4. `SLAVEOF` 设置主从

Redis Sentinel 主故障自动切换

## 事务

`MULTI-EXEC` 事务只能保证原子性，不保证可见性，故存在并发事务问题

`watch`乐观锁


## SUB/PUB

发布/订阅，通过 channel 来实现。

实现一个 redis datasource，实时同步数据。如果只是 value 类型的数据，通过简单的 S/P channel 即可拿到变更的值，其中不设置 key 这个概念。
但是， 如果数据为 Hash 类型，如果用以上的channel简单做法，有两种方式： 1. hash value 全部更新，无论是否发生变化; 2. 针对发布的消息做定制化处理，如"{"key" : "code1", "value" : "rule1"}”, 订阅方解析数据进行局部更新。

当然有更为麻烦的做法：[键空间通知（keyspace notification）](http://redisdoc.com/topic/notification.html)。这个是借助于 channel 来实现的，但是传递的消息不是直接数据。 `keyspace` 会传递对 key 进行的操作信息，如 set, del 等；而 `keyevent` 会传递操作 key 的名称。首先客户端连接要设置 `notify-keyspace-events` 决定传递哪些类型的通知。

如果要做 datasource, 则需要解析传递的数据，并查询 redis 更新本地数据，但是这样只是设计 key 级别的更新。好处就是不用而外额外手动更新 redis 数据了（当然可以在更新本地数据的时再更新 redis）。





















