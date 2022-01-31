## Java
- 使用yum安装（也可以自己下载安装到指定文件夹）
```
1.查看yum原中是否有想关套件
yum -y list java*

2.安装
yum -y install 对应版本

3.修改/etc/profile并执行source /etc/profile

export JAVA_HOME=/usr/share/jdk1.6.0_14  
export PATH=$JAVA_HOME/bin:$PATH  
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

```
## MySQL
## Redis
- 下载对应压缩包
- - `yum install gcc-c++`--一定要先安装这个
- 进入文件夹
	- `make && make install`--默认编译在`usr/local/bin`中。`make`是编译，`make install`是安装
		- 注意在`redis-5.0.8`文件夹中`make`，在`/redis-5.0.8/src`下`make install`
	- `make install PREFIX=/usr/local/redis`指定编译路径
	- 将`redis`解压缩后的文件中的配置文件复制到指定文件件，这里我们复制到`/usr/local/bin`中:`cp /opt/redis-5.0.8/redis.conf /usr/local/bin/redisconf/`
- 默认不是后台启动的：将配置文件中的`daemonize`选项改为`yes`,来后台启动
- 启动`redis`服务":在`bin`目录下运行`redis-server redisconf/redis.conf`，且指定了配置文件
- 客户端连接redis：`redis-cli -p 6379`
- 查看`redis`进程是否开启：`ps -ef|grep redis`