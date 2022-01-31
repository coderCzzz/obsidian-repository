## 服务器启动源码
- 剖析到`Netty`调用`doBind`方法，追踪到`NioServerSocketChannel`的`doBind`
- `Debug`程序到`NioEventLoop`类的`run`代码
- 重点看`bossGroup`和`workerGroup`
	- `bossGroup`用于接受Tcp请求，它会将请求交给`workerGroup`，`workerGroup`会获取到真正的连接，然后和连接进行通信
	- `EventLoopGroup`是事件循环组，含有多个`EventLoop`，可以注册`channel`，用于事件循环中去进行选择
	- 会创建`EventExecutor`数组：`children=new EventExecutor[nThreads]`,每个元素的类型都是`NIOEeventLoop`，`NIOEventLoop`实现了`EventLoop`接口和`Executor`接口。
- `try`块中创建了一个`ServerBootStrap`对象，它是一个引导类，用于启动服务器和引导整个程序的初始化，和`ServerChannel`关联，而`ServerChannel`继承了`Channel`

## 接收请求源码

## pipeline源码

## channelHandler源码

## Netty心跳源码

## EventLoop源码

## 任务加入线程池源码