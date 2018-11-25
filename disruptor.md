disruptor

# disrutor 为什么这么快？


## ringbuffer 

1. 无锁，使用CAS，锁的效率比CAS低
2. 避免了“伪共享”，避免了无关缓存行的刷新
3. 循环队列，减少了GC
