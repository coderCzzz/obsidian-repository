## 编码和解码的基本介绍
- 编写网络应用程序时，因为数据在网络中传输的都是二进制字节码数据，在发送数据时就需要编码，接收数据时就需要解码
- `codec`编码器的组成部分有两个：`decoder`解码器和`encoder`编码器。`encoder`负责把业务数据转换成字节码数据，`decoder`负责把字节码数据转换成业务数据

## Netty本身的编码解码机制以及相关问题
- Netty本身提供了一些`codec`编解码器
- 编码器
	- `StringEncoder`对字符串数据进行编码
	- `ObjectEncoder`对Java对象进行编码
- 解码器
	- `StringDecoder`对字符串数据进行解码
	- `ObjectDecoder`对Java对象进行解码
- Netty本身的底层使用的仍是Java序列化技术，而Java序列化技术存在如下问题
	- 无法跨语言
	- 序列化后体积太大，是二进制编码的5倍多
	- 序列化性能太低

## Protobuf
### 简介
- `Google`开源项目，一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化或序列化。适合做数据存储或`rpc`数据交换格式。
- 支持跨平台、跨语言，即客户端服务器端可以使用不同语言编写。
- 高性能、高可用性
- 使用`protobuf`编译器能自动生成代码，`Protobuf`将类的定义使用`.proto`文件进行描述。
- 然后使用`protoc.exe`编译器根据`.proto`自动生成java文件
- 使用示意图
![[截屏2021-12-08 下午4.22.53.png]]

### Protobuf实例-生成类
- 客户端发送一个`Student PoJo`对象到服务器（通过`Protobuf`编码）
- 服务器接收`Student PoJo`对象，并显示信息（通过`Protobuf解码`）
#### 步骤
- 引入`Protobuf`包
```
	<dependency>  
 <groupId>com.google.protobuf</groupId>  
 <artifactId>protobuf-java</artifactId>  
 <version>3.6.1</version>  
</dependency>
```
- 编写对应`.proto`文件
```
syntax = "proto3"; // 协议版本  
option java_outer_classname="StudentPoJo"; //生成的外部类名，同时也是文件名  
message Student { // 会在StudentPoJo外部类生成一个内部类Student，它是真正发送的pojo对象  
 int32 id = 1; // Student类中有一个属性名字为id，类型为int32（proto类型） 1表示属性序号，不是值  
 string name = 2;  
}
```
- 使用`protoc.exe`编译对应`.proto`文件生成对应java文件
```
protoc.exe --java_out=. Student.proto
```
- 发送对象
```
// 发送一个Student 对象到服务器  
StudentPoJo.Student s = StudentPoJo.Student.newBuilder().setId(4).setName("诸葛亮").build();  
ctx.writeAndFlush(s);
```
- 记得在`pipleline`中添加编解码器
```
pipeline.addLast("encoder", new ProtobufEncoder());  
// 需要指定对哪一种对象进行解码  
pipeline.addLast("decoder", new ProtobufDecoder(StudentPoJo.Student.getDefaultInstance));
```

### 传输多种类型
#### 要求
- 客户端可以随机发送`Student POJO/Worker POJO`对象到服务器
- 服务器能够接收并判断是哪种数据，并显示信息

#### 步骤
- 编写对应`.proto`文件并编译
```
syntax = "proto3"; // 协议版本  
option optimize_for = SPEED; // 加速解析  
option java_package= "com.atguigu.netty.codec2"; //指定生成到哪个包下  
option java_outer_classname="MyDataInfo"; //生成的外部类名，同时也是文件名  
  
//protobuf可以使用message管理其他的message  
message MyMessage {  
    // 定义一个枚举  
 enum DataType{  
        StudentType = 0; // proto3要求enum的编号从0开始  
 WorkerType = 1;  
 }  
  
    // 用data_type标识传的是哪一个枚举类型  
 DataType data_type = 1; // 表示一个属性是DataType  
  
 // 表示每次枚举类型最多只能出现其中的一个，节省空间  
 oneof dataBody{  
        Student student=2; //第二个属性 ，但是只能2 3 选一个  
 Worker worker = 3; //第三个属性  
 }  
}  
  
  
message Student { // 会在StudentPoJo外部类生成一个内部类Student，它是真正发送的pojo对象  
 int32 id = 1; // Student类中有一个属性名字为id，类型为int32（proto类型） 1表示属性序号，不是值  
 string name = 2;  
}  
message Worker{  
    string name = 1;  
 int32 id = 2;  
}
```
- 发送对象
```
// 随机发送Student或Worker对象
int random = new Random().nextInt(3);
MyDataInfo.MyMessage myMessage = null;
if (random ==0) {
	myMessage = MyDataInfo.MyMessage.newBuilder().setDataType(MyDataInfo.MyMessage.DataType.StudentType).setStudent(MyDataInfo.Student.newBuilder().setId(5).setName("赵云").builder).builder();
}
ctx.writeAndFlush(myMessage);
```
- 在`pipeline`设置编解码器