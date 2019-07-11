# CompletableFuture

### 异步转同步？

```java
long time = System.currentTimeMillis();
CompletableFuture<String> f0 = CompletableFuture.supplyAsync(()->{
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("enter 0 " + Thread.currentThread().getName());
    return "A";
}).thenApply(s->{
    System.out.println("enter 1 " + Thread.currentThread().getName());
    return s + "B";
}).thenApply(s->{
    System.out.println("enter 2 " + Thread.currentThread().getName());
    return s + "C";
});
System.out.println("spent " + (System.currentTimeMillis() - time) / 1000);
System.out.println(f0.get());
```
按此运行，结果：

```
spent 0
enter 0 ForkJoinPool.commonPool-worker-1
enter 1 ForkJoinPool.commonPool-worker-1
enter 2 ForkJoinPool.commonPool-worker-1
ABC
```
当将sleep操作注释后：

```
enter 0 ForkJoinPool.commonPool-worker-1
enter 1 main
enter 2 main
spent 0
ABC
```
注释后，看起来异步转同步了，这个现象是怎么产生的？

结果输出对比：

* sleep后的操作是由线程池中的线程执行的，而无sleep后的操作是由主线程执行的

### 猜测

`supplyAsync`虽然是异步的，但是后续的`thenApply`是依赖于上一个`CompletableFuture`是否完成。当`supplyAsync`立即完成，由主线程触发`thenApply`，否则由`supplyAsync`的执行线程触发

### 源码

`supplyAsync`方法异步执行`AsyncSupply`

```java
// AsyncSupply#run()
public void run() {
	// d: supplyAsync返回值
    CompletableFuture<T> d; Supplier<T> f;
    if ((d = dep) != null && (f = fn) != null) {
        dep = null; fn = null;
        // 还未执行完成
        if (d.result == null) {
            try { 
            	// 记录执行结果
                d.completeValue(f.get());
            } catch (Throwable ex) {
                d.completeThrowable(ex);
            }
        }
        // 通知等待起结果的action并执行
        d.postComplete();
    }
}
```

`CompletableFuture#postComplete()`

```java
final void postComplete() {
    CompletableFuture<?> f = this; Completion h;
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
    	// 替换 stack head
        if (f.casStack(h, t = h.next)) {
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                h.next = null;    // detach
            }
            // 
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}
```

何时将CompletableFuture放进stack

`CompletableFuture#thenApply`

```java
private <V> CompletableFuture<V> uniApplyStage(
    Executor e, Function<? super T,? extends V> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<V> d =  new CompletableFuture<V>();
    // uniApply 如果上个操作已完成（result != null）则进行f.apply(result). 否则将操作压栈
    if (e != null || !d.uniApply(this, f, null)) {
        UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
        // 压栈
        push(c);
        c.tryFire(SYNC);
    }
    return d;
}
```

回到`h.tryFire`，则调用`UniApply#tryFire`

```java
final CompletableFuture<V> tryFire(int mode) {
    CompletableFuture<V> d; CompletableFuture<T> a;
    if ((d = dep) == null ||
        !d.uniApply(a = src, fn, mode > 0 ? null : this))
        return null;
    dep = null; src = null; fn = null;
    return d.postFire(a, mode);
}
```

此次调用`d.uniApply`必然返回true，因为src(上一个操作)已经完成了(result != null)。此方法则返回

剩余的`thenApply`也是这样

## 结论

解释下“异步转同步”的现象，未注释sleep时,由于`thenApply`是同步操作且依赖与上一个异步操作`AsyncSupply`的结果，操作压栈等待上一个操作完成并触发，故而操作异步执行；而注释后，`AsyncSupply`立即完成，对于下一个操作`thenApply`则不需要压栈等待，直接在主线程中运行，故而看起来同步执行









