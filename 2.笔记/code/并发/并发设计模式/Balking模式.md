## 引言
```
class AutoSaveEditor{
	// 文件是否被修改过
	boolean changed = false;
	// 定时任务线程池
	ScheduledExecutorService 	ses=
	Executors.newSingleThreadScheduledExecutor();
	// 定时执行自动保存

void startAutoSave(){

ses.scheduleWithFixedDelay(()->{

autoSave();

}, 5, 5, TimeUnit.SECONDS);

}

// 自动存盘操作

void autoSave(){

if (!changed) {

return;

}

changed = false;

// 执行存盘操作

// 省略且实现

this.execSave();

}

// 编辑操作

void edit(){

// 省略编辑逻辑

......

changed = true;

}

}
	
}
```
可以看出上面的代码是线程不安全的，因为对共享变量`changed`的读写没有使用同步。
最简单的解决方法是对使用`changed`的地方加锁。

- 可以看出，`changed`就是一个状态变量，当状态变量符合某个条件时，执行某个业务逻辑，相当于if，放在多线程下，就是多线程版本的if。
- **Balking模式**：多线程版本if

## Balking模式经典实现
```
boolean changed=false;

// 自动存盘操作

void autoSave(){

synchronized(this){

if (!changed) {

return;

}

changed = false;

}

// 执行存盘操作

// 省略且实现

this.execSave();

}

// 编辑操作

void edit(){

// 省略编辑逻辑

......

change();

}

// 改变状态

void change(){

synchronized(this){

changed = true;

}

}
```

## volatile实现Balking模式--需要对原子性无要求

