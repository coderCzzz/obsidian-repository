# 一主多从数据同步
数据往主数据库写，mysql底层通过同步复制，将数据备份到从库。
延伸知识：
[[24.主备一致--binlog格式]]
[[25.高可用-主备延迟]]
[[26.备库为什么延迟好几个小时--备库并行复制能力]]
[[27.主库出问题了，从库怎么办]]
[[28.读写分离有哪些坑]]

## 一主多从配置
- 打开多个数据库配置文件`my.ini`,写入以下配置
```
[mysqld]
server-id=1000 //服务器id。服务器id要不同
log-bin=mysql-bin //主从复制使用的文件名
```
- 配置主服务器
	- 配置管理用户，用于从节点连接主节点使用--多从就是这里多配置几个用户
		- `create user '主服务器用户名'@'从服务器ip' IDENTIFIED by '主服务器密码'`
	- 为从服务器授予复制权限--多从，这里给每个从服务器授权
		- `grant relication slave on *.* to '主服务器用户名'@'从服务器ip'`
	- 激活权限
		- `flush PRIVILEGES`
	- 可选：`show master status`查看配置成功情况
		- 显示信息
			- `File`:保存的日志文件
			- `Position`:日志文件的偏移量
		- 可以使用`show binlog events in 'mysql-bin.000001'`查看二进制日志信息
- 配置从属服务器-多从，给每个从服务器配置
	- 指向要连接的主服务器
		```
		CHANGE MASTER TO
		MASTER_HOST='主服务器ip',MASTER_PORT='主服务器端口',
		MASTER_USER='主服务器用户名',MASTER_PASSWORD='主服务器密码',
		MASTER_LOG_FILE='日志文件mysql-binlog.000001',
		MASTER_LOG_POS=951//即上面show master status显示的偏移量，从这个位置开始复制
		```
	- 开始进行主从复制
		- `start slave`
	- 可选：查看从属服务器状态`show slave status`
		- 主要看`Slave_IO_Running`和`Slave_SQL_Running`

# Sharding JDBC

## 概述
- java框架，可以看做增强型的JDBC
- 不是做分库分表的
- 主要做两个功能：数据分片、读写分离
	- 简化对分库分表后数据相关操作

## 数据库的水平分表环境搭建

- 引入依赖
- 编写相关配置
	- 配置数据源
	- 指定表分布情况
	- 指定主键增长机制
	- 指定分片策略
	- 打开sql输出日志-可选