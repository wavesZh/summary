# http

## SocketTimeout/ConnectionRequestTimeout/ConnectTimeout

* SocketTimeout: read timeout，等待数据的超时时间，两个连续数据包之间的最大不活动周期
* ConnectionRequestTimeout: 从连接池中获取连接的超时时间
* ConnectTimeout: 连接建立超时时间

那是否会出现尝试从连接池中获取连接，发现无空闲连接，然后建立连接的情况吗？