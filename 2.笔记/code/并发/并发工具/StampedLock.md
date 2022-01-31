java1.8中提供与读写锁相同的功能，但是更快--StampedLock

## StampedLock支持的三种锁模式
- 写锁
- 悲观读锁
- 乐观读--实际上是无锁操作
写锁、悲观读锁类似于`ReadWriteLock`的写锁和读锁，不同的是，`StampedLock`写锁和悲观读锁加锁成后，会返回一个`stamp`，解锁需要传入该`stamp`
- `ReadWriteLock`的读写锁是互斥的，但是`StampedLock`的乐观读，多个线程读的时候，允许一个线程获取写锁。
- `StampedLock`乐观读也会返回一个`stamp`，由于是无锁操作，在进入临界区时，需要使用对应的校验方法`validate(stamp)校验`，校验失败需要升级为悲观读锁
```
class Point {

private int x, y;

final StampedLock sl =

new StampedLock();

// 计算到原点的距离

int distanceFromOrigin() {

// 乐观读

long stamp =

sl.tryOptimisticRead();

// 读入局部变量，

// 读的过程数据可能被修改

int curX = x, curY = y;

// 判断执行读操作期间，

// 是否存在写操作，如果存在，

// 则 sl.validate 返回 false

if (!sl.validate(stamp)){

// 升级为悲观读锁

stamp = sl.readLock();

try {

curX = x;

curY = y;

} finally {

// 释放悲观读锁

sl.unlockRead(stamp);

}

}

return Math.sqrt(

curX * curX + curY * curY);

}

}
```

### 注意
- StampedLock的功能是`ReadWriteLock`的子集
- 不可重入
- 悲观读锁、写锁都不支持条件变量
-  **StampedLock 一定不要调用中断操作，如果需要支持中断功能，一定使用可中断的悲观读锁 readLockInterruptibly() 和写锁 writeLockInterruptibly()**