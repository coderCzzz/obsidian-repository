# redo log
## 为什么需要redo log
- 主要是让数据库有崩溃恢复的能力
- 我们知道，数据库查询或修改数据，是先在内存中的`buffer pool`中的页进行修改，再在合适的时机将其刷新的磁盘。--[[CheckPoint技术]]
- 因此，假如在没有刷新到磁盘时，数据库宕机，这部分数据就会丢失，我们需要将这些修改记录到一个文件中，这个文件就是`redo log`。
- 但是将`redo log`写入到磁盘，不是同样的要消耗IO资源吗？我们要`redo log`是为了解决数据在内存中可能导致的不安全，做个备份，而数据放在缓冲区是减小数据读取IO，两者相比，数据放在缓冲区得到的利益更多。

## crash-safe与Write-Ahread Log
- `crash-safe` 因为有了`redo log`，MySQL可以保证数据库出现异常，数据不丢失，这种能力称为`crash-safe`
- `Write-Ahread Log`
	- 先写日志，在写磁盘。我认为是因为修改了数据并不是马上刷新到磁盘上，才会有先写日志再写磁盘这一顺序的。
	- 并且如果马上写是随机访问，而根据`redo log`去更新磁盘中的数据，就将随机读写变成了顺序读写。
## redo log概述
### 修改一条数据的过程
- 先看缓冲池中有没有对应的页，如果有，则直接修改。没有的话，将对应的数据页读取到缓冲池并修改。
- 将上述操作写入到`redo log`缓冲池，再在一定的时机，将`redo log`缓冲池里的缓冲刷新到`redo log`。如果启用了`binlog`，这里使用两阶段写，将`redo log`的状态置为`prepare`。写`binlog`后，将`redo log`置为`commit`状态。

### redo log block
- `redo log`记录的是xxx表空间的xxx数据页xxx偏移量做了xxx操作
- `redo log`按块，一块一块写入到磁盘
- ![[截屏2021-11-22 下午4.33.39.png]]、
	- `Header`
		- `block no`：`log buffer`类似一个数组，`block`是一个个元素，该字段标识了在这个数组中的位置
		- `data length`：`block`中写入的字节数据数量
		- `first record group`：标识`log block`中第一个日志的偏移量
			- 同一个事务产生的`block`做一个`group`，如可能T1事务产生的数据刚好填满一个`block`+另一个`block`的一半，则另一个事务T2从另一个`block`的一半以下接着写，这个数值就是标识这个一半往下的位置的。
		- `checkpoint on`：最后被写入时检查点第4字节的值。

### redo log buffer
- `redo log buffer`在内存池中，会划分出多个`redo log block`。
- 占用一块连续的内存空间，默认大小16MB。可以使用`innodb_log_buffer_size`动态调整
- `redo log buffer`何时写入到磁盘，也就是何时往磁盘写`redo log`
	- 事务提交时写入
		- 并行事务提交，事务A执行到一半，写了一些到`redo log buffer`，此时事务B提交，且`innodb_flush_log_at_commit`为1。事务B将`redo log buffer`的日志全部持久化到磁盘，会带上事务A的日志。
	- `redo log buffer`使用量达到了参数`innodb_log_buffer_size`的一半，这种只`write`，不`fsync`
	- `Master Thread`每隔1s会将`redo log block`刷新到磁盘
	- MySQL关闭

### `redo log`刷盘机制
- write与fsync
	- Write：将数据从内存写入到系统缓存
	- fsync: 将数据从系统缓存写入到磁盘
- 如何控制事务提交时，`redo log`的刷盘行为
	- `innodb_flush_log_at_trx_commit`控制
		- 0 ：事务提交时不进行写入重做日志。只依靠`Master Thread`每秒进行，且使用`fsync`持久化到磁盘
		- 1 ：每次事务提交，需要`write`和`fsync`
		- 2: 只将`redo buffer`中的数据刷进系统缓存，即`write`，不进行`fsync`。

### 组提交
- 最开始只有`redo log`的组提交，后面是`redo log`和`binlog`一起的组提交，因为两阶段提交的存在。最开始只有`redo log`组提交时，binlog开启两阶段提交时，组提交会失效。
- ![[截屏2021-11-23 下午4.58.39.png]]
- 事务按照提交的顺序进入队列，最先进入队列的事务成为`leader`
- `Flush`阶段
	- 将当前队列中的所有事务的`binlog`统一写入内存
- `Sync`阶段
	- 将当前队列中所有事务的`binlog`通过一次`fsync`统一写入到磁盘中
- `commit`阶段
	- 由队列中的`leader`根据一定顺序调用存储引擎层的事务提交，完成两阶段的第二阶段提交
- `binlog_max_flush_queue_time`控制`Flush`阶段的等待时间
- `binlog_group_commit_sync_delay`表示延迟多少微秒才调用`fsync`
- `binlog_group_commit_sync_no_delay_count`表示累计多少次以后才`fsync`

### redo log file group
- 由N个大小相同的`redo log`组成一个`redo log file group`，N默认为2
- `redo log`采用追加写，当写满时，触发`Checkpoint`，再清除一部分记录接着写。
- 因此`redo log file`的大小很关键，过小会导致循环切换`redo log file`与`Checkpoint`。

# binlog
## binlog概述
- 为什么需要`binlog`
	- 让`MySQL`具有搭建集群、数据备份、恢复数据的能力
- 在`MySQL`的`Server`层产生
- 可以使用`show variables like '%datadir%'`查看存放位置
- 主要有两种文件：`mysql-bin.0000xx`和`mysql-bin.index`
	- `mysql-bin.0000xx`保存着对`MySQL`更改的逻辑
	- `mysql-bin.index`保存前者的保存位置
- binlog主要记录：对xxx表中id=yyy行做了什么修改，修改后的值是什么
- 不会记录`select show`等操作。

## binlog写入机制
- 一个事务的binlog不能拆分，必须一次性写入。
### binlog高速缓存
- 所有未`commit`的事务产生的`binlog`，都会被先记录到`binlog`的高速缓存中。等事务`commit`，再将缓存中的数据写入到`binlog`日志文件。
- 高速缓存大小由`binlog_cache_size`控制，默认32768
	- 不能太大，浪费内存
	- 不能太小，当一个事务产生的日志超过该参数，会将缓存中的binlog数据写到临时文件中
- 每个线程都有自己的`binlog cache`，但是写入同一份binlog文件


### 刷盘机制
- 由`sync_binlog`控制
	- 0：不会主动将binlog落盘。仅将binlog写入到系统缓冲中
	- 1：事务提交时，将binlog刷新到磁盘
	- N：N大于1时，表示开启组提交。每次提交事务都`write`，但是累计N个事务后才`fsync`

### binlog格式
- 三种格式：`statement` 、 `row` 、`mixed`
[[24.主备一致--binlog格式]]


## redo log与binlog的区别

## 答疑
- 一些结论
	- redo log是用来崩溃恢复的
	- 两阶段提交是为了保证主备数据一致
	- binlog是用来归档的
### 两阶段提交的不同瞬间，MySQL发生异常重启，如何保证数据完整性
![[截屏2021-11-23 上午11.19.40.png]]
- `commit`的理解
	- 不是`commit`命令，指的是事务提交的最后一步
	- `commit`语句执行，包含`commit`步骤。

#### 时刻A发生崩溃
- 此时binlog还没写，redolog也没提交，因此崩溃恢复，该事务会回滚

#### 时刻B发生崩溃
- 崩溃恢复逻辑
	- 如果redo log里的事务是完整的，即有了`commit`标识，则直接提交
	- 如果只有完整的`prepare`，则判断事务的`binlog`是否完整
		- 是，提交事务--2a
		- 不是，回滚事务。
- 时刻B对应即2a的情况，崩溃恢复会提交事务

#### MySQL如何知道binlog是完整的
- 根据binlog的格式信息
	- `statement`格式，最后会有COMMIT
	- `row`格式，最后会有一个`XID event`
- `5.6.2`版本后，还有`binlog-checksum`参数验证`binlog`正确性

#### redo log和binlog如何关联
- 有一个共同字段XID，崩溃恢复时，会按顺序扫描redo log
	- 如果碰到既有prepare，又有commit的redo log，直接提交
	- 碰到只有`prepare`的redo log，则拿着XID去binlog找对应的事务。

#### 既然`prepare`的`redo log`和完整的`bin log`已经能重启恢复，为什么还要设计`commit`，也就是两阶段提交
- 保证主备数据一致
- `binlog`已经写入，则会被从库使用
- 因为需要保证主库中这个事务也提交

#### 那么先redo log在binlog，恢复的时候需要两个日志都完整，这种和两阶段提交的逻辑是否一样
- `redo log`提交完成后，事务不能回滚。如果此时binlog写入失败，就会导致数据域binlog不一致

#### binlog支持崩溃恢复吗
- 不支持
- ![[截屏2021-11-23 下午3.14.53.png]]
	- 如图，此时`crash`，假如binlog2写完了，但是未commot，重启后事务会回滚，然后使用binlog2恢复。对于binlog1认为提交完整，不会再应该用一次。
	- 由于先写日志再写磁盘，可能binlog1在崩溃时还没持久到磁盘，且binlog中没有记录数据页的更新细节，也就是xxx表空间的xxx页xxx偏移量做了xxx操作
	- 实际上是binlog无法判断哪些被持久到磁盘，哪些没有被持久到磁盘。而redo log中的记录被持久到磁盘会才会被删除。

#### 数据写入后的最终落盘，是从redo log更新过来的还是从`bugger pool`更新过来的
- 刷脏页过程与`redo log`毫无关系
- 崩溃恢复时，如果判断一个数据页可能丢失了更新，会将它读到内存，然后让`redo log`更新内存内容

#### 先写redo log buffer，还是先写redo log
- `insert`或`update`时，先修改内存数据，在`redo log buffer`中写入日志
- 真正将日志写到`redo log`文件，是在执行commit语句时。