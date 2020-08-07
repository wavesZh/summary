HttpServer.create() -> HttpServerBind.handle -> HttpServerHandle.bind -> HttpServerHandle.tcpConfiguration								

ReactorHttpHandlerAdapter

child-obs: ChildObserver -> HttpServerHandle


HttpServerHandle -> HttpServerBind

child-handler: BootstrapInitializerHandler -> Http1Initializer
												-> HttpServerCodec
												-> AccessLogHandler
												-> SimpleCompressionHandler
												-> HttpTrafficHandler
												-> HttpServerMetricsHandler
												-> ChannelOperationsHandler

head 靠近 io 层
inbound: head -> last     
outbound: last -> head


channelActive
-> ChannelOperationsHandler: ReactorNetty.CONNECTION ChannelOperations -> SimpleConnection


channelRead
-> HttpTrafficHandler: ReactorNetty.CONNECTION HttpServerOperations -> ChannelOperations -> SimpleConnection
-> ChannelOperationsHandler: HttpServerOperations.onInboundNext -> HttpServerHandle


HttpServerHandle.onStateChange : subscribe
-> ReactorHttpHandlerAdapter : doOnError(), doOnSuccess()
-> HttpWebHandlerAdapter 
	-> ExceptionHandlingWebHandler: error(), onErrorResume()
	-> FilteringWebHandler
	-> DispatcherHandler: 



## spring 5.2.3 org.springframework.http.HttpHeaders#clearContentHeaders

https://github.com/spring-projects/spring-framework/commit/f9c1565f4ef031904c3545a70e63080aeade6e90

Prior to this commit, when WebFlux handlers added `"Content-*"` response
headers and an error happened while handling the request, all those
headers would not be cleared from the response before error handling.

This commit clears those headers from the response in two places:
* when invoking the handler and adapting the response
* when writing the response body

Not removing those headers might break HTTP clients since they're given
wrong information about how to interpret the HTTP response body: the
error response body might be very different from the original one.

	
