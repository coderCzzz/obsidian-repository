## InnoDB逻辑存储结构
### 组成
![[截屏2021-11-18 下午3.48.52.png]]
- [[#表空间]]
	- [[#段]]
	- [[#区]]
	- [[#页]]

### 表空间
- 默认情况下有一个共享表空间`ibdata1`，所有数据都存放在这个表空间内。
- 设置参数`innodb_file_per_table`，每个表生成一个表空间
	- 但是仅存放数据、索引、插入缓冲bitmap页
	- 其他数据如回滚信息、插入缓冲索引页、系统事务信心、二次写缓冲等还是放在原来的共享表空间

### 段
- 表空间由多个段组成
	- 数据段--即B+树的叶子节点
	- 索引段-非叶子节点
	- 回滚段

### 区
- 由连续的页组成，每个区的大小为1MB。
- 为了保证页的连续性，每次从磁盘申请4-5个人区。
### 页
- 磁盘管理最小单位。默认大小为16KB，后面有压缩页技术。
- 常见页类型
	- 数据页
	- undo页
	- 系统页
	- 事务系统页
	- 插入缓冲位图页
	- 插入缓冲空闲列表页
	- 未压缩的二进制大对象页
	- 压缩的二进制大对象页

