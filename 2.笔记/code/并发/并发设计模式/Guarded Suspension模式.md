## 引言
有一个东西，我们需要等待它的结果，然后再去做某些事。类似于吃饭预定包间，去的时候大堂经理通知还没收拾好包间，等包间收拾好了，大堂经理通知过去就餐。
## Guarded Suspension模式
![[截屏2022-01-13 下午4.20.49.png]]
- `GuardedObject`对应大堂经理，内部有一个成员变量-受保护对象，对应包间。
	- 两个成员方法
	- `get(Predicate<T> p)`:对应就餐,就餐需要的条件是包间收拾好，p就是用来描述这个前置条件。
	- `onChanged(T obj)`方法对应服务员收拾包间，可以`fire`一个事件，而这个事件可以改变前提条件p的计算结果。
- 上图绿色线程对应需要就餐的顾客，蓝色线程对应收拾包间的服务员。

## Guarded Suspension内部实现
- 实际上是管程
- `get()`方法通过条件变量的`await()`方法实现等待
- `onChanged()`通过条件变量的`signalAll()`方法实现唤醒功能
```
class GuardedObject<T>{
	//受保护对象
	T obj;
	final Lock lock = new ReentrantLock();
	final Condition done = lock.newCondition();
	final int timeout = 1;
	
	// 受保护对象
	T get(Predicate<T> p) {
		lock.lock();
		try{
			while(!p.test(obj)) {
				done.await(timeout, TimeUnit.SECONDS);
			}
		}catch(InterruptedException e) {
			throw new RuntimeException(e);
		}finally{
			lock.unlock();
		}
		return obj;
	}
	
	//事件通知方法
	void onChanged(T obj) {
		lock.lock();
		try{
			this.obj = obj;
			done.signalAll();
		}finally{
			lock.unlock();
		}
	}
}

```
