## 使用CompletableFuture优化泡茶程序--优化异步编程
```
package org.fenixsoft.compile;  
  
  
import java.util.concurrent.CompletableFuture;  
import java.util.concurrent.ExecutionException;  
import java.util.concurrent.TimeUnit;  
  
/**  
 * @author caoz  
 * @date 2022/1/8 10:46 上午  
 **/public class CompleteFutureDemo {  
  
    public static void main(String[] args) throws ExecutionException, InterruptedException {  
        // 任务1：洗水壶->烧开水  
 CompletableFuture<Void> f1 = CompletableFuture.runAsync(()->{  
            System.out.println("T1:洗水壶...");  
 sleep(1);  
 System.out.println("T1：烧开水...");  
 sleep(15);  
 });  
 // 任务2：洗茶壶->洗茶杯->拿茶叶  
 CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {  
            System.out.println("T2:洗茶壶...");  
 sleep(1);  
 System.out.println("T2:洗茶杯...");  
 sleep(1);  
 System.out.println("T2:拿茶叶...");  
 sleep(1);  
 return "龙井";  
 });  
 // 任务3：泡茶  
 CompletableFuture<String> f3 = f1.thenCombine(f2, (s, tf) -> {  
            System.out.println("T1:拿到茶叶" + tf);  
 System.out.println("T1：泡茶...");  
 return "上茶"+tf;  
 });  
 System.out.println(f3.get());  
 }  
  
    static void sleep(int time) {  
        try {  
            TimeUnit.SECONDS.sleep(time);  
 } catch (InterruptedException e) {  
            e.printStackTrace();  
 }  
    }  
}
```

## CompletableFuture的使用
### 创建CompletableFuture对象
4个静态方法
- 使用默认线程池
	- 默认情况下使用公共的`ForkJoinPool`线程池，默认创建的线程数是CPU的核数
```
runAsync(Runnable runnable) 无返回值
supplyAsync(Supplier<U> supplier)有返回值
```
- 指定线程池参数的
```
static CompletableFuture<Void>

runAsync(Runnable runnable, Executor executor)

static <U> CompletableFuture<U>

supplyAsync(Supplier<U> supplier, Executor executor)
```

## CompletionStage 接口
- 任务之间有不同的时序关系：如串行、并行、汇聚。
- CompletionStage可以清楚的描述任务之间的时序关系
- CompletionStage也可以方便的描述异常处理

### 描述串行关系
- `thenApply`既能接收参数，也能返回值
- `thenAccept`支持参数，不返回值
- `thenRun`既不接收参数，也不支持返回值
- `thenCompose`新创建一个子线程
- 加`Async`代表异步执行
```
CompletionStage<R> thenApply(fn);

CompletionStage<R> thenApplyAsync(fn);

CompletionStage<Void> thenAccept(consumer);

CompletionStage<Void> thenAcceptAsync(consumer);

CompletionStage<Void> thenRun(action);

CompletionStage<Void> thenRunAsync(action);

CompletionStage<R> thenCompose(fn);

CompletionStage<R> thenComposeAsync(fn);
```
#### 代码示例
```
CompletableFuture<String> f0 =

CompletableFuture.supplyAsync(

() -> "Hello World") //①

.thenApply(s -> s + " QQ") //②

.thenApply(String::toUpperCase);//③

System.out.println(f0.join());

// 输出结果

HELLO WORLD QQ
```

### 描述AND汇聚关系
```
CompletionStage<R> thenCombine(other, fn);

CompletionStage<R> thenCombineAsync(other, fn);

CompletionStage<Void> thenAcceptBoth(other, consumer);

CompletionStage<Void> thenAcceptBothAsync(other, consumer);

CompletionStage<Void> runAfterBoth(other, action);

CompletionStage<Void> runAfterBothAsync(other, action);
```

### 描述OR汇聚关系
```
CompletionStage applyToEither(other, fn);

CompletionStage applyToEitherAsync(other, fn);

CompletionStage acceptEither(other, consumer);

CompletionStage acceptEitherAsync(other, consumer);

CompletionStage runAfterEither(other, action);

CompletionStage runAfterEitherAsync(other, action);
```

### 异常处理
```
CompletionStage exceptionally(fn);

CompletionStage<R> whenComplete(consumer);

CompletionStage<R> whenCompleteAsync(consumer);

CompletionStage<R> handle(fn);

CompletionStage<R> handleAsync(fn);
```

