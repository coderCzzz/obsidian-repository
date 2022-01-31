## 信号量模型
计数器和等待队列对外透明，也就是说外界不知道他们的存在，只能通过提供的三个方法来访问。
- 一个计数器
- 一个等待队列
- 三个方法。都是原子性的，`Semaphore`保证这三个方法都是原子操作。
	- `init()`设置计数器的初始值
	- `down()`计时器减1，如果此时计数器的值小于0，则当前线程被阻塞，否则当前线程可以继续执行
	- `up()`：计数器加1，如果此时计数器的值小于等于0，则唤醒等待队列中的一个线程，并将其从等待队列中移除。

## 如何使用
- `acquire`与`release`对应`down`和`up`
```
static int count;

// 初始化信号量

static final Semaphore s

= new Semaphore(1);

// 用信号量保证互斥

static void addOne() {

s.acquire();

try {

count+=1;

} finally {

s.release();

}

}
```

## 信号量模型实现限流器
`Semaphore`相比于`Lock`，不仅仅是实现互斥锁，更能实现允许多个线程访问一个临界区。--对应即各种池化资源。

```
class ObjectPool<T, R> {

	final List<T> pool;
	final Semaphore sem;
	
	ObjectPool(int size, T t) {
		pool = new Vector<T>(){};
		for (int i =0; i<size; i++) {
			pool.add(t);
		}
		sem = new Semaphore(size);
	}
	
	// 利用线程池对象，调用func
	R exec(Function<T, R> func) {
		T t = null;
		sem.acquire();
		try{
			return func.apply(t);
		}finally{
			pool.add(t);
			sem.release;
		}
	}
	
	// 创建线程池
	ObjectPool<Long, String> pool = new ObjectPool<Long, String>(10, 2);
}
```