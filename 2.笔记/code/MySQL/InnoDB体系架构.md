## InnoDB体系架构
![[截屏2021-11-17 上午9.16.07.png]]
- 后台线程
	- 作用
		- 刷新内存池中数据
		- 将已修改的数据文件刷新到磁盘文件
	- 组成
		- [[Master Thread工作方式]]
		- [[#IO Thread]]
		- [[#Purge Thread]]
		- [[#Page Cleaner Thread]]
- 内存池
	- 作用
		- 维护多个内部数据结构
		- 缓存磁盘上的数据，同时对在磁盘文件的数据修改之前在这里缓存
		- 重做日志(redo log)缓存
	- 组成
		- [[#缓存池]]
		- [[#重做日志缓冲]]
		- [[#额外的内存池]]

## 内存池
### 内存中的数据对象
![[截屏2021-11-17 上午9.24.55.png]]
- 缓冲池
	- 缓冲池中包括数据页、索引页、插入缓冲、自适应哈希等等
	- 允许有多个缓冲池实例，每个页根据哈希值平均分配到不同的缓冲池中
		- 可以通过`innodb_buffer_pool_instances`进行数量配置
- 重做日志缓冲
- 额外内存池

### 缓存池
- 数据库进行读取页操作时，先看页是否在内存池中，如果没有，则从磁盘中读取并缓存在内存中，如果在缓存中，则直接读取。
- 进行修改操作，先修改缓冲池中的页，再以一定的频率刷新到磁盘
	- 通过[[CheckPoint技术]]刷回磁盘
- 因此一般需要设置较大的缓冲池大小
	- 可以通过参数`innodb_buffer_pool_size`设置
#### 缓冲池内存管理
##### 基本算法
- 使用LRU算法，即最频繁使用的页放在LRU列表的前端，最少使用的页在LRU列表的尾端
- 当缓冲池中不能存放新读取的页时，先释放尾端的页。
##### 优化
- midpoint位置：新读取的页不是直接放在首部，而是放在midpoint的位置
	- midpoint默认在LRU列表的5/8处。由参数`innodb_old_blocks_pct`（默认37）控制
	- `midpoint`位置之后的称为`old`列表，之前的称为`new`列表
- 为什么要这么做？
	- 因为如果将数据放在头部，扫描多个数据时，就可以从尾部一个个释放，导致之前的热点数据被刷出
- `innodb_old_blocks_time`表示页读取到mid位置后需要等待多久才会被加入到LRU的热端。--保证热点数据不被刷出。
- `page made young`：页从LRU的old部分别移动到new部分
- `page not made young`:由于设置了`innodb_old_blocks_time`未被移动的操作

##### Free列表
- 数据库刚启动时，LRU列表为空，此时页都存放在Free列表中
- 当需要从缓存池中分页时，先查看Free列表中是否有空闲页，如果有从Free列表中删除，并加入LRU列表。否则就根据LRU算法分配。

##### Flush列表
- 当缓冲池中数据被修改时，此时页的数据和磁盘中的数据不一致，该页被称为脏页
- Flush列表中的页即为脏页
- LRU列表和Flush列表中都有脏页。

##### 查看LUR列表和Free列表情况
![[截屏2021-11-17 上午9.47.01.png]]
- `buffer_pool_size`:缓冲池中存在的页
- `Free_buffers`：空闲列表中存在的页
- `Database pages`:LRU列表中的页
- 相加不相等的原因是因为缓存池中还有自适应哈希、Lock信息等
- `pages made youngs`表示页从old移动到new次数
- `buffer pool hit rate`：缓冲池的命中率，一般应大于95%

##### 页压缩与管理
- InnoDB引擎从1.0支持压缩页功能，将原本16K的页压缩成1、2、4、8KB
- 非16KB的页通过`unzip_LRU`列表管理。（LUR列表包含unzip_LRU列表的页）
- `unzip_LRU`内存分配
	- 对不同压缩页大小的页进行分别管理
	- 通过伙伴算法进行内存分配：如申请页为4KB大小时
		- 检查4KB的`unzip_LRU`列表是否有空闲页
		- 有直接用，没有就检查8KB的`unzip_LRU`列表
		- 若能得到空闲页，将页分成两个4KB页，存放到4KB的`unzip_LRU`
		- 不能，则申请一个16KB的页，将页分为一个8KB，两个4KB分别存放到对应的`unzip_LRU`中

### 重做日志缓冲
- 先将重做日志信息放入重做日志缓冲，再以一定的频率刷新到重做日志文件
- 一般每秒都会将缓冲刷新到日志
- 重做日志缓冲大小通过`innodb_log_buffer_size`控制，默认为8MB
- 什么时候刷盘
	- `Master Thread`每一秒将重做日志缓冲刷新到重做日志文件
	- 每个事务提交都会将刷盘
	- 重做日志缓冲剩余空间小于1/2时刷盘。

### 额外的内存池
- 缓存池中的某些数据结构进行内存分配时，需要从额外的内存池中申请，如缓冲帧、缓冲控制对象等。

## 后台线程
### Master Thread
- 负责将缓冲池中的数据异步刷新到磁盘，保证数据一致性
- 包含脏页的刷新、合并插入缓冲、undo页的回收。
[[Master Thread工作方式]]

### IO Thread
- InnoDB大量使用AIO处理IO请求
- `IO Thread`主要负责这个IO请求的回调处理。
- 1.0版本前共4个IO Thread
	- write
	- read
	- insert buffer
	- log
- 1.0.x版本后，read和write增大到4个，使用以下参数控制
	- `innodb_read_io_threads`
	- `innodb_write_io_threads`

### Purge Thread
- 事务提交后，undolog可能不在需要，`Purge Thread`用来回收使用并分配的undo页
- 1.1版本前，`purge`操作只在`Master Thread`中进行
- 1.1版本后，在单独的线程中进行

### Page Cleaner Thread
- 1.2.x版本引入，将之前脏页刷新等操作放在单独的线程

