# Zookeeper


## Watches

特点： watcher 是一次性用品，每次触发完后，就失效了，需要重新注册

* 一次性触发，每次触发后失效，如需持续生效，需要重复注册
* 


### watch 机制是否可靠

[Zookeeper 通知更新可靠吗？ 解读源码找答案！](https://cloud.tencent.com/developer/article/1158972)


在 sessionTime

没有发送确认机制，即使通知客户端失败，也不会重复通知

在网络错误的情况下，客户端与服务端断开连接，watch event 无法到达客户端，需要客户端手动读取数据已保证最新数据，但可能会出现 “ABA” 问题，丢失中间事件。


