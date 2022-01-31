## 什么是Checkpoint技术
- 将缓冲池中脏页刷回到磁盘
- 两种
	- Sharp Checkpoint
	- Fuzzy CheckPoint

## 什么时候触发Checkpoint
- Sharp Checkpoint发生在数据库关闭时或切换要写的`redo log`，将**所有的脏页**都刷新回次磁盘
	- 默认工作方式`innodb_fast_shutdown=1`
- 但数据库运行时使用Fuzzy Checkpoint，只刷新部分脏页。触发时机如下
	- `Master Thread Checkpoint`
		- 每秒或每10秒从缓冲池的脏页列表中刷新一定比例的页回磁盘--异步
	- `FLUSH_LUR_LIST Checkpoint`
		- 保证LUR列表中有一定数量的空闲页可供使用
		- 通过参数`innodb_lru_scan_depth`控制，默认1024
	- `Async/Sync Flush Checkpoint`
		- 重做日志不可用时（满了），强制将一些页刷新回磁盘，此时脏页是从脏页列表中选取的
		- [[#重做日志的Checkpoint机制详解]]
	- `Dirty Page too mush Checkpoint`
		- 脏页太多强制Checkpoint，刷新一部分脏页到磁盘。由`innodb_max_dirty_pages_pct`控制百分比，默认为75%。

## 解决了什么问题
- 缩短数据库恢复时间
	- 崩溃恢复就是重启时照着`redo log`中最后一次`Checkpoint`之后的日志回放一遍
	- 只用binlog当然可以恢复数据库，但是要从头开始，时间太久
	- 数据库宕机，不需要重做所有日志，`Checkpoint`之前的页都已被刷到磁盘。
- 缓冲池不够用时，将脏页刷新到磁盘
	- 缓冲池不够用，淘汰尾部的页，假如该页为脏页，需要将脏页刷到磁盘。
- 重做日志不可用时，刷新脏页


## 重做日志的Checkpoint机制详解
- InnoDB引擎通过`LSN(Log Sequence Number)`标记版本。
- 若将已经写入到重做日志的LSN记为`redo_lsn`，已经刷回磁盘最新页的LSN记为`checkpoint_lsn`，定义`checkpoint_age`=`redo_lsn`-`checkpoint_lsn`
- 再定义`async_water_mark = 75% *total_redo_log_file_size`;`sync_water_mark=90% *total_redo_log_file_size`
- 若每个重做日志文件大小为1G，定义2个重做日志文件，则从总大小为2GB，则此时
`async_water_mark=1.5GB`,`sync_water_mark==1.8GB`，则 
- 当`checkpoint_age < async_water_mark`时，不需要刷新任何脏页到磁盘
- 当`async_water_mark < checkpoint_age < sync_water_mark`，触发`Asnyc Flush`，从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足`checkpoint_age < async_water_mark`
- 当`sync_water_mark < checkpoint_age`,触发`Sync Flush`，从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足`checkpoint_age < async_water_mark`

## LSN
- 序列号。每次操作都会导致序列号增加。
- 表空间中的数据页、缓存页、内存中的`redo log`、磁盘中的`redo log`、`checkpoint`都有`LSN`标记。
- 比如MySQL重启时可以对比数据页和`redo log`的`LSN`的大小，如果前者的比后者的小，说明数据页中缺失了一部分数据。