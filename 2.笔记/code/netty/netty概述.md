## 原生NIO存在的问题
1. NIO的类库和API繁杂，使用麻烦，需要熟练掌握`Selector`、`SelectorSocketChannel`、`SocketChannel`、`ByteBuffer`等
2. 需要具备其他额外技能：如要熟练java多线程与网络编程
3. 开发工作量和难度都非常大，如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络阻塞和异常流的处理等等
4. JDK NIO的bug：如`Epoll Bug`会导致`Selector`空轮训，最终导致`CPU`100%等

## 现有线程模型
- 目前存在的线程模型有
	- 传统阻塞IO模型
	![[截屏2021-12-06 上午10.16.18.png]]
	- `Reactor`模式
		- 基于IO复用模型：多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理。
		- 基于线程池复用线程资源：不必为每个连接创建线程，将==连接完成后==的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务
		- ![[截屏2021-12-07 上午8.15.38.png]]
		- 核心组成
			- `Reactor`: `Reactor`在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对IO事件作出反应。
			- `Handlers`: 处理程序执行IO事件要完成的实际事件。`Reactor`通过调度适当的处理程序来响应IO事件，处理程序执行非阻塞操作。
- 根据`Reacotr`的数量和处理资源池的数量不同，有3种典型的实现
	- 单`Reactor`单线程
	- 单`Reactor`多线程
	- 主从`Reactor`多线程--netty
- Netty线程模型（基于主从`Reactor`多线程模型做了一定的改进，其中主从`Reactor`多线程模型有多个`Reactor`）

### 单Reactor单线程
![[截屏2021-12-07 上午8.23.34.png]]
可以看出`Reactor`和`Handler`在同一个线程，在高并发的时候，表现会差。
- 优点：模型简单，没有多线程、进程通信、竞争的问题，全部在一个线程完成
- 缺点：
	- 性能问题：只有一个线程，无法完全发挥多核CPU的性能。`Handler`在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈
	- 可靠性问题：线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。
- 使用场景：客户端的数量有限，业务处理非常快速，比如`Redis`在业务处理的时间复杂度O(1)的情况。

### 单Reactor多线程
- 通过`read`方法将数据读入后，将`Handler`对应的业务处理放到线程池，让work线程池分配一个线程去处理，`Handler`只对对应事件作出反应，不做业务处理。
- ![[截屏2021-12-07 上午8.35.00.png]]
- 优点：可以充分的利用多核CPU的处理能力
- 缺点：多线程数据共享和访问比较复杂，`Reactor`处理了所有的事件的监听和响应，在单线程运行，在高并发场景容易出现性能瓶颈

### 主从Reactor多线程
![[截屏2021-12-07 上午8.49.38.png]]
- 优点：父线程与子线程的数据交互简单、职责明确，父线程只需要接收新连接，子线程完成后续的业务处理
- 缺点：编码复杂度高
- 实例：`Nginx`主从`Reactor`多进程模型，`Memcached`主从多线程，`Netty`主从多线程模型

## Netty模型
### 简单版
![[截屏2021-12-07 上午8.59.19.png]]

### 进阶版
![[截屏2021-12-07 上午9.01.03.png]]
### 详细版
![[截屏2021-12-07 上午9.12.55.png]]

## 实战
- 创建Netty项目，Netty服务器在6668端口监听，客户端能发送消息给服务器。服务器可以恢复消息给客户端

### 服务器端
服务器
```
package com.atgui.netty.simple;  
  
import io.netty.bootstrap.ServerBootstrap;  
import io.netty.channel.ChannelFuture;  
import io.netty.channel.ChannelInitializer;  
import io.netty.channel.ChannelOption;  
import io.netty.channel.nio.NioEventLoopGroup;  
import io.netty.channel.socket.SocketChannel;  
import io.netty.channel.socket.nio.NioServerSocketChannel;  
  
/**  
 * @author caoz  
 * @date 2021/12/7 9:21 上午  
 **/
 public class NettyServer {  
  
    public static void main(String[] args) throws InterruptedException {  
        // 创建bossgroup和workergroup  
 		// 说明：1.创建两个线程组bossGroup和WorkerGroup 
		// 2.bossGroup只处理连接请求，真正和客户端业务处理会交给workerGroup 
		// 3.两个都是无限循环 
		NioEventLoopGroup bossGroup = new NioEventLoopGroup();  					NioEventLoopGroup workerGroup = new NioEventLoopGroup();  
  
 	try{  
        // 创建服务器端的启动对象，配置参数  
 		ServerBootstrap bootstrap = new ServerBootstrap();  
  
 		// 使用链式编程来设置  
 		bootstrap.group(bossGroup, workerGroup) //设置两个线程组  
 		.channel(NioServerSocketChannel.class) //使用NioSocketChannel作为服务器的通道实现  
 		.option(ChannelOption.SO_BACKLOG, 128) //设置线程队列得到连接个数  
 		.childOption(ChannelOption.SO_KEEPALIVE, true) // 设置保持活动连接状态  
 		.childHandler(new ChannelInitializer<SocketChannel>() { //创建一个通道测试对象  
 	// 给pipeline设置处理器 @Override  
 			protected void initChannel(SocketChannel ch) throws Exception {  
                            ch.pipeline().addLast(new NettyServerHandler());  
 						}  
             }); //给workerGroup的EventLoop对应的管道设置处理器  
  
 		System.out.println("....服务器 is ready....");  
  
 		// 绑定端口并且同步，生成一个ChannelFuture对象（后面会讲）  
 		// 这里已经启动了服务器 ChannelFuture cf = bootstrap.bind(6668).sync();  
  
 		// 对关闭通道进行监听  
 		cf.channel().closeFuture().sync();  
 		}finally {  
            bossGroup.shutdownGracefully();  
 			workerGroup.shutdownGracefully();  
 		}  
    }  
}
```
处理器
```
package com.atgui.netty.simple;  
  
import io.netty.buffer.ByteBuf;  
import io.netty.buffer.Unpooled;  
import io.netty.channel.ChannelHandlerContext;  
import io.netty.channel.ChannelInboundHandlerAdapter;  
  
import java.nio.ByteBuffer;  
import java.nio.charset.StandardCharsets;  
  
/**  
 * @author caoz  
 * @date 2021/12/7 9:41 上午  
 **/  
/**  
 * 说明 * 1.我们自定义一个Handler 需要继承netty规定好的某个HandlerAdapter */
 public class NettyServerHandler extends ChannelInboundHandlerAdapter {  
  
    /**  
 * * 读取数据实际（可以读取客户端发送的消息） * 1. ChannelHandlerContext 上下文对象，含有管道pipleline， 通道channel， 地址 * 2. msg：客户端发送的数据，默认Object * * */ @Override  
 	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        System.out.println("server ctx = "+ ctx);  
  
 		// 将msg转成一个Bytebuf,netty提供，不是NIO的ByteBuffer  
 		ByteBuf buf = (ByteBuf) msg;  
 		System.out.println("客户端发送消息时：" + 	buf.toString(StandardCharsets.UTF_8));  
 		System.out.println("客户端地址："+ctx.channel().remoteAddress());  
 }  
  
    // 数据读取完毕  
 	@Override  
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {  		
		// write+flush,将数据写入到缓存，并刷新  
		 // 一般对发送的数据进行编码 
		 ctx.writeAndFlush(Unpooled.copiedBuffer("hello, 客户端", CharsetsUtil.UTF_8));  
 }  
  
    // 异常处理，一般是关闭通道  
 	@Override  
 	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws 	Exception {  
       ctx.channel().close();  
 	}  
}
```

### 客户端
客户端
```
package com.atgui.netty.simple;  
  
import io.netty.bootstrap.Bootstrap;  
import io.netty.channel.ChannelFuture;  
import io.netty.channel.ChannelInitializer;  
import io.netty.channel.nio.NioEventLoopGroup;  
import io.netty.channel.socket.SocketChannel;  
import io.netty.channel.socket.nio.NioSocketChannel;  
  
/**  
 * @author caoz  
 * @date 2021/12/7 10:31 上午  
 **/
 public class NettyClient {  
  
    public static void main(String[] args) throws InterruptedException {  
        // 客户端只需要一个事件循环组  
 		NioEventLoopGroup eventExecutors = new NioEventLoopGroup();  
 		try{  
            // 创建客户端启动对象，客户端使用的不是ServerBootstrap,而是Bootstrap  
 			Bootstrap bootstrap = new Bootstrap();  
 			// 设置相关参数  
 			bootstrap.group(eventExecutors) //设置线程组  
 			.channel(NioSocketChannel.class) //设置客户端通道实现类  
 			.handler(new ChannelInitializer<SocketChannel>() {  
                        @Override  
 					protected void initChannel(SocketChannel ch) throws Exception 	{  
                            ch.pipeline().addLast(new NettyClientHandler()); //加入自己的处理  
 				}  
                    });  
  
 			// 启动客户端去连接服务器  
 			ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 6668).sync();  
 			// 给关闭通道进行监听  
 			channelFuture.channel().closeFuture().sync();  
 		}finally {  
            eventExecutors.shutdownGracefully();  
 		}  
    }  
}
```
处理器
```
package com.atgui.netty.simple;  
  
import io.netty.buffer.ByteBuf;  
import io.netty.buffer.Unpooled;  
import io.netty.channel.ChannelHandlerContext;  
import io.netty.channel.ChannelInboundHandlerAdapter;  
import io.netty.util.CharsetUtil;  
  
/**  
 * @author caoz  
 * @date 2021/12/7 10:39 上午  
 **/
 public class NettyClientHandler extends ChannelInboundHandlerAdapter {  
  
    // 当通道就绪时就会触发该方法  
 	@Override  
 	public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("client "+ctx);  
 		ctx.writeAndFlush(Unpooled.copiedBuffer("hello, server ", CharsetUtil.UTF_8));  
 }  
  
    // 当通道有读取事件时，会触发  
 	@Override  
 	public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        ByteBuf byteBuf = (ByteBuf) msg;  
 		System.out.println("服务器回复的消息: "+byteBuf.toString(CharsetUtil.UTF_8));  
 		System.out.println("服务器的地址：" +ctx.channel().remoteAddress());  
 }  
  
    // 异常处理  
 	@Override  
 	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {  
        cause.printStackTrace();  
 		ctx.close();  
 }  
}
```


### 代码分析
- `BossGroup`和`WorkerGroup`含有的`NioEventLoop`线程数默认为`CPU核数*2`，子线程可以在`children`属性中看到