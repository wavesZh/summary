# distributed transaction

## seate

### link

* [https://mp.weixin.qq.com/s/EzmZ-DAi-hxJhRkFvFhlJQ](分布式事务中间件Fescar-RM模块源码解读)

### 模块

* TM(Transaction Manager): 全局事务管理器。负责全局事务的生命周期：开启，提交，回滚等
* RM(Resource Manager): 资源管理器。负责分支事务：分支注册、状态汇报、接收 TC 指令，驱动分支（本地）事务提交回滚等
* TC(Transaction Coordinator): 事务协调器。协调全局事务的提交回滚等

### 二阶段提交协议

1. 一阶段 prepare

TM 开启全局事务，通知各 RM，RM 执行本地事务但不commit，并 report 处理结果至 TM

2. 二阶段 ready

TM 只要收到处理失败信息则通知各 RM 回滚，除非收到全部处理成功信息才提交全局事务。完成后释放锁


### AT

Automatic (Branch) Transaction Mode. 有点隐式事务的味道，不需要自定义 commit/rollback 逻辑，业务无侵入

分支事务依赖本地事务的保障， 如果数据库不支持 ACID 事务，则需要使用 MT 模式


### RM

JDBC 数据源代理是 RM 实现的基础




