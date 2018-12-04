# JMM

Java内存模型杂记

## JMM中8种原子性的操作

| opt | scope | detail |
| ---------- | :-----------:  | :-----------: |
|lock(锁定)| 主内存 | 将一个变量标示一个线程独占状态 |
|unlock(解锁) |主内存| 将锁定状态的变量释放，以便其他线程锁定|
|read(读取)|主内存|将主内存中变量传输到工作内存中，以便load|
|load(载入)|工作内存|将read传输的变量放入工作内存中的变量副本中|
|use(使用)|工作内存|将变量副本中的值传输给执行引擎|
|assign(赋值)|工作内存|将从引擎接收的值放入工作内存的变量副本中|
|store(存储)|工作内存|将变量副本传输到主内存中，以便wirte|
|write(写入)|主内存|将store传输的变量放入主内存中的变量中|


## volatile

