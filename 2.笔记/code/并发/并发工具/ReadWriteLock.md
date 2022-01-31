Java SDK并发包的多种工具，应对不同场景。

适用于读多写少并发场景`ReadWriteLock`

## 什么是读写锁
所有的读写锁都遵循一下三条基本原则：
1. 允许多个线程同时读写共享变量
2. 只允许一个线程写共享变量
3. 如果一个写线程正在执行写操作，此时禁止读线程读共享变量。

**读写锁允许多个线程同时读共享变量，互斥锁不允许。**

## 使用读写锁快速实现一个缓存
```
class Cache<K, V> {
	final Map<K, V> m = new HashMap<>();
	
	final ReadWriteLock rwl = new ReentrantReadWriteLock();
	// 读锁
	final Lock r = rwl.readLock();
	// 写锁
	final Lock w = rwl.writeLock();
	
	// 读缓存
	V get(K key) {
		r.lock();
		try{
			return m.get(key);
		}finally{
			r.unlock();
		}
	}
	
	// 写缓存
	V put(Stirng key, Data v) {
		w.lock();
		try{
			return m.put(key, v);
		}finally{
			w.unlock();
		}
	}
}
```

## 读写锁的升级与降级
- 不允许升级，允许锁的降级
- 实现了`Lock`接口，因此也支持`tryLock()` `lockInterruptibly()`等方法
- 只有写锁支持条件变量，读锁不支持条件变量。
