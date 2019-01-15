# spi

service provide interface，服务发现**机制**
。通过接口编程。实现服务可插拔。

## 基本组成结构
 
1. api接口

2. factory或者mananger等，负责找到具体服务的提供者

3. 服务提供者


## spi , rpc , rmi 

1. rpc(Remote Procedure Call Protocol): 远程过程调用协议，通过网络从远程计算机上请求调用某种服务。**网络服务协议**。 

2. rmi(Remote Method Invocation): 远程方法调用。仅限于java。

## jdk和spring boot中的spi

1. jdk: `ServiceLoader`, "META-INF/services/具体的实现类 包名+类名"

2. spring boot: `SpringFactoriesLoader`, "META-INF/spring.factories"

