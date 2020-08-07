# Reactor Core

关键词：观察者模式，函数式，异步化，背压

## 模型

### 观察者模式和订阅发布模式

观察者和订阅发布不是一个概念。

观察者模式组件：publisher, subscriber
订阅发布组件： publisher, topic, subscriber

对比可知，订阅发布多了一个组件 topic，使得 publisher 和 subscriber 解耦。

Reactor 则属于观察者模式。


观察者模式的变种： Publisher.subscribe(Subscriber)

publisher
    -> subscribe(Subscriber)
        -> onSubscribe(Subscribtion)
            -> request(n)
                -> onConsume, onComplete, onError


## 函数式




## 背压






