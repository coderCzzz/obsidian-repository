# Sharding Sphere
## 结构
- 核心功能
	- 数据分片--分表
	- 分布式事务
	- 数据库治理
	- 多模式连接
	- 管控界面
- 接入端
	- Sharding-JDBC--java独有的，切入程序
	- Sharding-Proxy--类似nginx中间件代理，不局限于语言
	- Sharding-Sidecar
- 开放生态

## Sharding JDBC
- 在java JDBC层提供额外服务
- 使用客户端直连服务器，以jar包形式提供服务
- 可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架

## 分库分表配置
- 引入依赖
- 配置
	- 配置数据源-多个数据库
	- 配置分片数据源
		- 通知数据源
		- 分片规则
- 分布式主键
	- 内置主键生成器
		- UUID
		- SNOWFLAKE
		- LEAF
		- 时钟回拨