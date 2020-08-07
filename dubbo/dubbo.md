# dubbo 杂记

说实话，感觉 Dubbo 的源码并好看，不优雅，感觉很混乱，除了 Spring 接入。
版本更迭与文档更新速度不匹配，不清楚新增的 feature 有什么应用场景。
Dubbo 感觉太重了。。。。

## provider 启动流程

1. exportServices

ServiceConfigBase#export
-> doExport
    -> doExportUrlsFor1Protocol：按照协议生成exporter
        -> ArgumentConfig 解析验证
        -> exportLocal
        -> RegistryProtocol#export
            -> doLocalExport -> ProtocolFilterWrapper#export：封装invoker -> DubboProtocol#export：启动server，监听
            -> getRegistry：创建Registry -> 注册provider
            

2. 调用链路：

NettyServerHandler#channelRead
-> MultiMessageHandler#received：处理多消息
    -> HeartbeatHandler#received：处理心跳请求
        -> AllChannelHandler#received：委托处理请求
            -> ChannelEventRunnable#run 
                -> DecodeHandler#recieved：解码 -> DecodeableRpcInvocation#decode
                -> HeaderExchangeHandler#received 正式开始处理 -> DubboProtocol$requestHandler#reply -> invoker.invoke



Filter请求链路：

EchoFilter
ClassLoaderFilter
GenericFilter
ContextFilter
TraceFilter
TimeoutFilter
MonitorFilter
ExceptionFilter
Invoker

## consumer

1. referServices

ReferenceConfigBase#get
-> createProxy
    -> ProtocolListenerWrapper#refer -> ProtocolFilterWrapper#refer -> RegistryProtocol#refer
        -> getRegistry：初始化Registry
        -> register ConsumerUrl：注册consumer信息
        -> RegistryDirectory#subscribe 
            -> FailbackRegistry#subscribe -> ZookeeperRegistry 查询 /dubbo/org.apache.dubbo.demo.DemoService/providers 是否有子节点
                -> RegistryDirectory#refreshOverrideAndInvoker：刷新invoker


2. 调用链路

InvokerInvocationHandler#invoke
-> AbstractProxyInvoker#invoke -> MockClusterInvoker#invoke 
    -> InterceptorInvokerNode#invoke -> ConsumerContextClusterInterceptor
    -> AbstractClusterInvoker#invoke：获取 invoker 列表，负载策略
        -> FailoverClusterInvoker#doInvoke：技术异常失败重试，调用其他invoker
            -> invoker.invoke：filter chain 处理
                -> ConsumerContextFilter 
                -> FutureFilter
                -> MonitorFilter
                -> GenericImplFilter
                -> AsyncToSyncInvoker -> DubboInvoker 
                ---- 网络层
                    -> request -> ReferenceCountExchangeClient -> HeaderExchangeClient -> HeaderExchangeChannel：构建DefaultFuture，timeoutCheck
                    -> send -> NettyClient -> NettyChannel


3. 响应链路

NettyClientHandler#channelRead
-> received -> NettyClient -> MultiMessageHandler -> HeartbeatHandler -> AllChannelHandler
    -> ChannelEventRunnable#run
        -> DecodeHandler#received -> DecodeableRpcResult#decode
            -> DefaultFuture#received


