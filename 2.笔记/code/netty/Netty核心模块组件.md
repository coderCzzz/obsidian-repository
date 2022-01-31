## Bootstrap与ServerBootstrap
1. 一个`Netty`应用通常由一个`Bootstrap`开始，主要作用是配置整个`Netty`程序，串联各个组件，`Netty`中的`Bootstrap`类是客户端程序的启动引导类，`ServerBootstrap`是服务端启动引导类
2. 常见方法
![[截屏2021-12-07 下午2.41.06.png]]
- 设置handler对应的是bossGroup，childHandler对应`workerGroup`

## Future和ChannelFuture
1. 通过`Future`和`ChannelFuture`可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件。
2. 常见方法
	1. `Channel channel()`：返回当前正在进行IO操作的通道
	2. `ChannelFutrue sync()`:等待异步操作执行完毕

## Channel
1. `Netty`网络通信的组件，能够用于执行网络IO操作
2. 通过Channel可获得当前网络连接的通道的状态与网络连接的配置参数
3. Channel提供异步的网络IO操作。
4. 调用立即返回一个ChannelFuture实例，通过注册监听器到ChannelFuture上，可以在IO操作成功、失败或取消时回调通知调用方
5. 支持关联IO操作与之对应的处理程序
6. 不同协议、不同阻塞类型的连接都有不同的Channel类型与之对应，常见Channel类型有
	1. NioSocketChannel：异步的客户端TCP Socket连接
	2. NioServerSocketChannel:异步的服务器端TCP Socket连接
	3. NioDatagramChannel：异步的UDP连接
	4. NioSctpChannel:异步的客户端Sctp连接
	5. NioSctpServerChannel：异步的Sctp服务器连接

## Selector
1. Netty基于Selector对象实现IO多路复用，通过Selector一个线程可以监听多个连接的Channel
2. 当向一个Selector中注册Channel后，Selector内部的机制就可以自动不断的查询这些注册的Channel是否有已就绪的IO事件。

## Handler

## Pipeline和ChannelPipeline
1. ChannelPipeline是一个`Handler`的集合，它负责处理和拦截入站inbound或者出栈outbound的事件和操作，相当于一个贯穿`netty`的链。也可以这样理解：ChannelPipeline是保存ChannelHandler的List，用于处理或拦截Channel的入站事件和出站操作
2. ChannelPipeline实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及Channel中各个ChannelHandler如何相互交互
3. 一个Channel对应一个ChannelPipeline
![[截屏2021-12-07 下午3.02.19.png]]

## ChannelHandlerContext
1. 保存Channel相关的所有上下文信息，同时关联一个ChannelHandler对象
2. 即ChannelHandlerContext中包含一个具体的事件处理器ChannelHandler,同时ChannelHandlerContext中也绑定了对应得`pipeline`和`Channel`的信息，方便对`ChannelHandler`进行调用
3. 常用方法
	1. `ChannelFuture close()`关闭通道
	2. `ChannelOutboundInvoker flsuh()`刷新
	3. `ChannelFuture writeAndFlush(Object msg)`，将数据写到`ChannelPipeline`中当前`ChannelHandler`的下一个`ChannelHander`开始处理

## ChannelOption
1. Netty在创建Channel实例后，一般需要设置ChannelOption参数
2. 常用参数如下
	1. ChannelOption.SO_BACKLOG:对应TCP/IP协议listen函数中的`backlog`参数，用来初始化服务器可连接队伍大小。服务器处理客户端连接请求是顺序处理的，所以同一时间只能处理一个客户端连接。多个客户端来的时候，服务器将不能处理的客户端连接请求放在队列中等待处理，backlog参数指定队列的大小
	2. ChannelOption.SO_KEEPALIVE：一直保持连接活动状态

## EventLoopGroup和实现类NioEventLoopGroup
1. EventLoopGroup是一组EventLoop的抽象，Netty为了更好的利用多核CPU资源，一般会有多个EventLoop同时工作，每个EventLoop维护着一个Selector实例。
2. EventLoopGrou提供next接口，可以从组里面按照一定规则获取其中一个EventLoop来处理任务。


## Unpooled类与应用实例
1. Netty提供的专门用来操作缓冲区的工具类
2. 常用方法
	- `public static ByteBuf copiedBuffer(CharSequence string, Charset charset)`

## 应用实例-群聊系统
### 要求
- `Netty`群聊系统，实现服务器端和客户端之间的数据简单通讯（非阻塞）
- 实现多人群聊
- 服务器端：可以检测用户上线、离线、并实现消息转发功能
- 客户端：通过`Channel`可以无阻塞的发送消息给其他用户，同时可以接受其他用户发送的消息（由服务器转发得到）

## Netty心跳检测机制
### 实例要求
- 编写一个`Netty`心跳检测机制，当服务器超过3秒没有读时，就提示读空闲
- 当服务器超过5s没有写操作时，就提示写空闲
- 服务器超过7秒没有读或写操作时，就提示读写空闲


## Netty通过WebSocket实现服务器和客户端的长连接
### 要求
- `Http`协议是无状态的，浏览器和服务器的请求相应一次，下一次会重新创建连接
- 要求：实现基于`webSocket`的长连接的全双工的交互
- 改变`Http`协议多次请求的约束，实现长连接，服务器可以发送消息给浏览器
- 客户端和服务器会相互感知，互相知道对方的关闭