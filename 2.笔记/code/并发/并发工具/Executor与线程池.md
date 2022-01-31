**创建线程的成本很高，应该避免频繁创建和销毁**

## Java线程池参数
- corePoolSize：线程池保持的最小线程数
- maximumPoolSize：线程池创建的最大线程数
- keepAliveTime：如果一个线程空闲了keepAliveTime，且线程池数此时大于corePoolSize，则该线程被回收
- workQueue：工作队列
- threadFactory：通过该参数可以自定义如何创建线程
- handler：自定义任务的拒绝策略。如果线程池中所有线程都在忙，且工作队列也满了，此时提交任务，线程池就会拒绝接收。
	- CallerRunsPolicy：提交任务的线程自己去执行任务
	- AbortPolicy：默认拒绝策略，抛出`RejectedExecutionException`
	- DiscardPolicy:直接丢弃任务，不抛出异常
	- DiscardOldestPolicy：丢弃最老的任务。

## 使用线程池需要注意什么
- 不建议使用`Executors`最重要的原因是`Executors`提供的很多方法默认使用无界的`LinkedBlockingQueue`，很容易导致OOM。
- 使用有界队列时，任务过多时，线程池会触发执行拒绝策略，线程池默认的拒绝策略会`throws RejectedExecutionException`，运行时异常，因此默认拒绝策略要慎重使用。
- 如果线程池处理的任务非常重要，建议定义自己的拒绝策略·。