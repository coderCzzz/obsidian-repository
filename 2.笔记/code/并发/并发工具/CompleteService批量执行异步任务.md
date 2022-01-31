## CompleteService
- CompleteService的实现原理内部维护了一个阻塞队列，当任务执行结束就把任务的执行结果加入到阻塞队列中(实际上是任务执行结果的Future对象加入到阻塞队列)

## 如何创建
- CompleteService接口的实现类是`ExecutorCompletionService`,两个构造方法


```
ExecutorCompletionService(Executor executor);
ExecutorCompletionService(Executor executor, BlockingQueue<Future<V>> completionQueue);
```
不指定`completionQueue`，会默认使用无界的`LinkedBlockingQueue`

## 代码示例
```
// 创建线程池

ExecutorService executor =

Executors.newFixedThreadPool(3);

// 创建 CompletionService

CompletionService<Integer> cs = new

ExecutorCompletionService<>(executor);

// 异步向电商 S1 询价

cs.submit(()->getPriceByS1());

// 异步向电商 S2 询价

cs.submit(()->getPriceByS2());

// 异步向电商 S3 询价

cs.submit(()->getPriceByS3());

// 将询价结果异步保存到数据库

for (int i=0; i<3; i++) {

Integer r = cs.take().get();

executor.execute(()->save(r));

}
```

## CompleteService接口说明
- 5个方法
```
Future<V> submit(Callable<V> task);

Future<V> submit(Runnable task, V result);

Future<V> take()

throws InterruptedException;

Future<V> poll();

Future<V> poll(long timeout, TimeUnit unit)

throws InterruptedException;
```

## CompletionService 实现 Dubbo 中的 Forking Cluster
- Dubbo 中有一种叫做**Forking 的集群模式**，这种集群模式下，支持**并行地调用多个查询服务，只要有一个成功返回结果，整个服务就可以返回了**。例如你需要提供一个地址转坐标的服务，为了保证该服务的高可用和性能，你可以并行地调用 3 个地图服务商的 API，然后只要有 1 个正确返回了结果 r，那么地址转坐标这个服务就可以直接返回 r 了。这种集群模式可以容忍 2 个地图服务商服务异常，但缺点是消耗的资源偏多

- 代码实现
```
/ 创建线程池

ExecutorService executor =

Executors.newFixedThreadPool(3);

// 创建 CompletionService

CompletionService<Integer> cs =

new ExecutorCompletionService<>(executor);

// 用于保存 Future 对象

List<Future<Integer>> futures =

new ArrayList<>(3);

// 提交异步任务，并保存 future 到 futures

futures.add(

cs.submit(()->geocoderByS1()));

futures.add(

cs.submit(()->geocoderByS2()));

futures.add(

cs.submit(()->geocoderByS3()));

// 获取最快返回的任务执行结果

Integer r = 0;

try {

// 只要有一个成功返回，则 break

for (int i = 0; i < 3; ++i) {

r = cs.take().get();

// 简单地通过判空来检查是否成功返回

if (r != null) {

break;

}

}

} finally {

// 取消所有任务

for(Future<Integer> f : futures)

f.cancel(true);

}

// 返回结果

return r;
```

