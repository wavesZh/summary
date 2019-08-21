## dubbo优雅停机

### link

[Dubbo 优雅停机](https://dubbo.apache.org/zh-cn/blog/dubbo-gracefully-shutdown.html)

### 定义

>优雅停机是指在停止应用时，执行的一系列保证应用正常关闭的操作。这些操作往往包括等待已有请求执行完成、关闭线程、关闭连接和释放资源等，优雅停机可以避免非正常关闭程序可能造成数据异常或丢失，应用异常等问题。优雅停机本质上是JVM即将关闭前执行的一些额外的处理代码


### 原理 

`DubboShutdownHook`负责销毁回收资源

```java
/**
 * Destroy all the resources, including registries and protocols.
 */
public void doDestroy() {
    if (!destroyed.compareAndSet(false, true)) {
        return;
    }
    // destroy all the registries
    AbstractRegistryFactory.destroyAll();
    // destroy all the protocols
    destroyProtocols();
}
```

`AbstractRegistryFactory.destroyAll();`主要负责清除 Registry 的注册和订阅信息以及关闭与 Registry 的连接

`destroyProtocols();`主要负责 Protocol 的销毁：

1. 取消 Protocol 所 export 和 refer 的 service
2. 释放资源。如连接等

### Registry 销毁

Registry 有多种实现，如Zookeeper,Etcd,Redis等，销毁时会调用起 `XXXRegistry#destroy()` 方法

```java
public void destroy() {
	// 清除注册信息
    super.destroy();
   	··· ···
   	// 一般为关闭连接
}
```
进入`FailbackRegistry#destroy()`，**Failback**表示故障自动恢复，那么会有个Timer用于定时失败补偿。那么需要关闭`retryTimer`

进入`AbstractRegistry#destroy()`

```java
public void destroy() {
    if (logger.isInfoEnabled()) {
        logger.info("Destroy registry:" + getUrl());
    }
    Set<URL> destroyRegistered = new HashSet<>(getRegistered());
    if (!destroyRegistered.isEmpty()) {
        for (URL url : new HashSet<>(getRegistered())) {
            if (url.getParameter(DYNAMIC_KEY, true)) {
                try {
                    unregister(url);
                    if (logger.isInfoEnabled()) {
                        logger.info("Destroy unregister url " + url);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to unregister url " + url + " to registry " + getUrl() + " on destroy, cause: " + t.getMessage(), t);
                }
            }
        }
    }
    Map<URL, Set<NotifyListener>> destroySubscribed = new HashMap<>(getSubscribed());
    if (!destroySubscribed.isEmpty()) {
        for (Map.Entry<URL, Set<NotifyListener>> entry : destroySubscribed.entrySet()) {
            URL url = entry.getKey();
            for (NotifyListener listener : entry.getValue()) {
                try {
                    unsubscribe(url, listener);
                    if (logger.isInfoEnabled()) {
                        logger.info("Destroy unsubscribe url " + url);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to unsubscribe url " + url + " to registry " + getUrl() + " on destroy, cause: " + t.getMessage(), t);
                }
            }
        }
    }
}
```
两个主要操作，`unregister` 和 `unsubscribe`。`unregister`将清除注册信息。`unsubscribe`清除订阅信息，关闭listener。前者可以看作为服务提供方，后者则为服务消费方

`unregister(url)` 会转到 `FailbackRegistry`，最终调用 `doUnregister(url)` 完成注册信息的清除工作

### Protocol 销毁

以常用的 `DubboProtocol` 为例

1. 关闭 server: 

* 发送只读事件至 client，使其停止使用与 server 相关的 channel 进行请求
* 先延时关闭server以处理已到达的请求

2. 关闭 client:

* 延时关闭client，channel

3. 清除 invoker

4. 清除 exporter
















