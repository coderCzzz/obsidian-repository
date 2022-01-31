## 引言
- `Java`的`String`类实现`replace()`方法时，没有改变原字符串中的`value[]`数组的内容，而是创建了一个字符串。本质上就是`Copy-On-Write`，也就是写时复制。
- 不可变对象的修改或写操作往往都是使用`Copy-On-Write`方法解决。

## Copy-On-Write模式应用领域
- `CopyOnWriteArrayList`和`CopyOnWriteArraySet`两个容器：实现的读操作都是无锁的。
- 操作系统领域：创建进程的API`fork()`。按需复制
- 函数式编程：函数式编程的基础是不可变性。因此所有修改操作都是`Copy-On-Write`