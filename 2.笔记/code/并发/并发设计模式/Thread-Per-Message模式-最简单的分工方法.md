## 前言
- **Thread-Per-Message**:委托他人代办，也就是每个任务分配一个独立的线程的模式。

## Thread实现Thread-Per-Message模式
经典应用场景：网络编程服务端的实现，为每个客户端请求创建一个独立的线程。当线程处理完请求后，自动销毁。
- 轻量级线程-协程：创建成本很低，大约跟创建一个普通对象的成本相似，创建速度很快。
- `OpenJDK`的`Loom`项目，解决`Java`语言的轻量级线程问题。该项目中轻量级线程被称为`Fiber`


## 用Fiber实现Thread-Per-Message
- 使用：将`new Thread(()->{}).start()`换成`Fiber.schedule(()->{})`即可

