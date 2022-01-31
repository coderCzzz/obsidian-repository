Java通过`ThreadPoolexecutor`提供的3个`submit()`方法和1个`FutureTask`工具类支持获得任务执行结果的需求

```
// 提交Runnable任务
Future<?> submit(Runnable task)
返回的Future仅可以用来断言任务已经结束

// 提交Callable任务
Future<T> submit(Callable<T> task);
这个Future可以调用get方法获取任务执行结果
//提交Runable任务及结果引用

<T> Future<T> submit(Runnable task, T result);
Future.get()的返回值会传给result
```

## Future接口
- 有5个方法，分别是
	- `cancel()`:取消任务
	- `isCancelled()`：判断任务是否已取消
	- `isDone()`：判断任务是否已结束
	- `get()`：获得任务执行结果
	- `get(timeout, unit)`：获得任务执行结果，支持超时机制
	- 两个获取任务执行结果的方法都都是阻塞式的，调用时如果任务还没执行完，则调用`get()`方法的线程会阻塞直到任务执行完才会被唤醒

## FutureTask工具类
- 两个构造函数
```
FutureTask(Callable<V> callable);
FutureTask(Runnable runnable, V result)
```
- 实现了`Runnable`和`Future`接口，因为实现了`Runnable`接口，因此可以作为任务提交给`ThreadPoolExecutor`去执行，也可以直接被`Thread`执行；又因为实现了`Future`接口，因此也能用来获得任务的执行结果。

## 实现最优泡茶
![[截屏2022-01-08 上午9.50.12.png]]
![[截屏2022-01-08 上午9.50.58.png]]
- 分析：首先进行分工，即高效地拆解任务并分配给线程。用两个线程T1、T2来模拟烧水泡茶程序，其中T1在执行到泡茶时需要等待T2完成拿茶叶的工序，等待这个动作，可以使用`Thread.join()、CountDownLatch`等实现，这里使用`Future`特性
- 创建两个`FutureTask`表示两个任务
```
package org.fenixsoft.compile;  
  
import java.util.concurrent.Callable;  
import java.util.concurrent.ExecutionException;  
import java.util.concurrent.FutureTask;  
import java.util.concurrent.TimeUnit;  
  
/**  
 * @author caoz  
 * @date 2021/12/28 9:45 下午  
 **/public class Test {  
    public static void main(String[] args) throws ExecutionException, InterruptedException {  
        FutureTask<String> ft2 = new FutureTask<>(new T2Task());  
 FutureTask<String> ft1 = new FutureTask<>(new T1Task(ft2));  
 Thread T1 = new Thread(ft1);  
 Thread T2 = new Thread(ft2);  
 T1.start();  
 T2.start();  
 System.out.println(ft1.get());  
  
 }  
  
  
  
  
  
}  
  
// T1Task需要执行的任务：洗水壶、烧开水、泡茶  
class T1Task implements Callable<String>{  
  
    FutureTask<String> ft2;  
  
 public T1Task(FutureTask<String> ft2) {  
        this.ft2 = ft2;  
 }  
  
    @Override  
 public String call() throws Exception {  
        System.out.println("T1：洗水壶...");  
 TimeUnit.SECONDS.sleep(1);  
  
 System.out.println("T1:烧开水...");  
 TimeUnit.SECONDS.sleep(15);  
  
 // 获取T2线程的茶叶  
 String tf = ft2.get();  
 System.out.println("T1:拿到茶叶："+tf);  
  
 System.out.println("T1:泡茶...");  
 return "上茶："+tf;  
 }  
}  
  
class T2Task implements Callable<String> {  
    @Override  
 public String call() throws Exception {  
        System.out.println("T2:洗茶壶...");  
 TimeUnit.SECONDS.sleep(1);  
  
 System.out.println("T2：洗茶杯...");  
 TimeUnit.SECONDS.sleep(2);  
  
 System.out.println("T2:拿茶叶...");  
 TimeUnit.SECONDS.sleep(2);  
 return "龙井";  
 }  
}
```
