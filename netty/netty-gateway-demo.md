# 踩坑


## Netty http server writeAndFlush 后，Client 依然等待


这个是 Http 协议的机制导致的。

1. close channel 会触发发送 last content，告知 client 响应结束。
2. 设置 content-length，让client判断是否已收到全部响应。


## Netty htttp client 发送请求后无响应

又是 Http 协议的坑，请求头 header 字段不能缺失，但值可以为空，否则会400




## hight-low write buffer watermark

只是提醒使用者，wirte buffer 达到高水位，最好不要在 write，没有实际的操作。避免出现 OOM. 