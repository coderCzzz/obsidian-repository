## 什么是分治
- 一种思想，把一个复杂的问题分解成多个相似的子问题，然后把子问题分解成更小的子问题，直到子问题简单到可以直接求解。

## Fork/Join
- Fork对应分治中的任务分解，Join对应的是结果合并。
- 计算框架主要包含两部分，分治任务的线程池`ForkJoinPool`和分治任务`ForkJoinTask`

### ForkJoinTask
- 抽象类，最核心的方法是`fork()`和`join()`
	- `fork()`会异步执行一个子任务，`join`则会阻塞当前的线程来等待子任务的执行结果。
- 有两个子类`RecursiveAction`和`RecursiveTask`。都定义了抽象方法`compute()`，区别是`RecursiveAction`定义的`compute()`无返回值。这两个子类也都是抽象类。

## 使用Fork/Join计算斐波那契数列
```
package org.fenixsoft.compile;  
  
import java.util.concurrent.ForkJoinPool;  
import java.util.concurrent.RecursiveTask;  
  
/**  
 * @author caoz  
 * @date 2022/1/9 8:55 上午  
 **/public class ForkJoinDemo {  
  
    public static void main(String[] args) {  
        // 定义分治任务线程池  
 ForkJoinPool fjp = new ForkJoinPool(4);  
 // 定义分治任务  
 Finbonacci fib = new Finbonacci(30);  
 // 启动分治任务  
 Integer result = fjp.invoke(fib);  
 System.out.println(result);  
 }  
  
    static class Finbonacci extends RecursiveTask<Integer> {  
        final int n;  
  
 Finbonacci(int n) {  
            this.n = n;  
 }  
  
        @Override  
 protected Integer compute() {  
            if (n <=1) {  
                return n;  
 }  
            Finbonacci f1 = new Finbonacci(n - 1);  
 f1.fork();  
 Finbonacci f2 = new Finbonacci(n-2);  
 return f2.compute()+f1.join();  
 }  
    }  
}
```

## ForkJoinPool工作原理
- 本质上是生产者-消费者实现，内部有多个任务队列，执行`invoke`或`submit()`提交任务时，按照一定的规则将任务提交到对应的任务队列
- 任务窃取：如果某个工作线程的任务队列空闲了，会窃取其他工作任务队列中的任务，这样就不会空闲了。
- 任务队列采用双端队列，工作线程正常获取任务和窃取任务从不同端消费。

## 模拟MapReduce统计单词数量
```
package org.fenixsoft.compile;  
  
import java.util.HashMap;  
import java.util.Map;  
import java.util.concurrent.ForkJoinPool;  
import java.util.concurrent.RecursiveTask;  
  
/**  
 * @author caoz  
 * @date 2022/1/9 9:19 上午  
 **/public class ForkJoinDemo2 {  
  
    public static void main(String[] args) {  
        String[] fc = {  
                "hello world", "hello me",  
 "hello forl", "hello join",  
 "fork join in world"  
 };  
 // 创建线程池  
 ForkJoinPool fjp = new ForkJoinPool(3);  
 // 创建任务  
 MR mr = new MR(fc, 0, fc.length);  
 Map<String, Long> result = fjp.invoke(mr);  
 result.forEach((k, v)->{  
            System.out.println(k+":"+v);  
 });  
  
 }  
  
    // MR模拟类  
 static class MR extends RecursiveTask<Map<String, Long>>{  
        private String[] fc;  
 private int start, end;  
  
 public MR(String[] fc, int fr, int to) {  
            this.fc = fc;  
 this.start = fr;  
 this.end = to;  
 }  
  
        @Override  
 protected Map<String, Long> compute() {  
            if(end - start == 1) {  
                return calc(fc[start]);  
 }else {  
                int mid = (start + end) /2;  
 MR mr1 = new MR(fc, start, mid);  
 mr1.fork();  
 MR mr2 = new MR(fc, mid, end);  
 return merge(mr2.compute(), mr1.join());  
 }  
        }  
  
        // 合并结果  
 private Map<String, Long> merge(Map<String, Long> m1, Map<String, Long> m2) {  
            Map<String, Long> result = new HashMap<>();  
 result.putAll(m1);  
 m2.forEach((k,v)->{  
                Long c = result.get(k);  
 if (c !=null) {  
                    result.put(k, c+v);  
 }else{  
                    result.put(k, v);  
 }  
            });  
 return result;  
 }  
  
        //统计单词数量  
 private Map<String, Long> calc(String line) {  
            HashMap<String, Long> result = new HashMap<>();  
  
 String[] words = line.split("\\s+");  
 for (String w: words) {  
                Long v = result.get(w);  
 if (v!=null) {  
                    result.put(w, v+1);  
 }else{  
                    result.put(w, 1L);  
 }  
            }  
            return result;  
 }  
    }  
  
  
  
}
```
