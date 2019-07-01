# 《Redis 实战》读书笔记


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

1. always。每个写命令立即同步。受硬盘性能限制且损伤硬盘
2. everysec。每秒同步。当硬盘忙于执行写操作，Redis 会自适应达到最佳写入速度
3. no。操作系统决定何时同步


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
* 配置 `save` 配置选项
* 关闭/终止服务（正常关闭前肯定是需要持久化，以便下次启动初始化数据）
* 主从服务器连接 `SYNC`

快照流程：

Redis 主进程 fork 出一个子进程将数据集进行快照并替换旧快照

fork 操作会阻塞 Redis。数据集越大，花费的时间越长


## 复制

高可用以及负载









