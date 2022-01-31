## 概述
- 任务队列中的Task有3种典型使用场景
	- 用户程序自定义的普通任务--举例说明
	- 用户自定义定时任务
	- 非当前`Reactor`线程调用`Channel`的各种方法
		- 例如在==推送系统==的业务线程里，根据==用户的标识==，找到对应的==Channel引用==，然后调用`Write`类方法向该用户推送消息，就会用到这种场景。最终的`Write`会提交到任务队列中被异步消费。

## TaskQueue自定义任务
- 假如服务器处理器`read`时有个耗时方法
- 注意加入到同一个`TaskQueue`的在同一个线程。因此如果下面的代码再加一个休眠20s的程序，则先执行休眠10s再执行休眠20s，而不是并发执行。
```
 @Override  
 public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception { 
  
 // 假如这里有个非常耗时的业务-->异步执行-->提交该channel对应的NioEventLoop的taskQueue
 // 解决方案1 用户程序自定义任务 
 	ctx.channel().eventLoop().execute(new Runnable() {  
            @Override  
 			public void run() {  
                // 这里写耗时任务  
 				try{  
                    Thread.sleep(10*1000);  
 					ctx.writeAndFlush(Unpooled.copiedBuffer("hello 客户端111",CharsetUtil.UTF_8));  
 				}catch (Exception e) {  
                    System.out.println("发生异常");  
 				}  
            }  
        }); 
		
	// 这里可以重复加任务
	ctx.channel().eventLoop().execute（）...
 }
```

### 用户自定义定时任务ScheduledTaskQueue
```
  @Override  
 public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {   
  
 // 假如这里有个非常耗时的业务-->异步执行-->提交该channel对应的NioEventLoop的taskQueue 
 // 解决方案1 用户程序自定义任务// 
 // 解决方案2：任务自定义定时任务->提交该channel对应的NioEventLoop的taskQueue 
 		ctx.channel().eventLoop().schedule(new Runnable() {  
            @Override  
 				public void run() {  
                try{  
                    ctx.writeAndFlush(Unpooled.copiedBuffer("hello ,5s后执行", CharsetUtil.UTF_8));  
 				}catch (Exception e) {  
                    System.out.println("发生异常");  
 				}  
            }  
        }, 5, TimeUnit.SECONDS);  
 }
```
