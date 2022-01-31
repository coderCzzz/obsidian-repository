# 设置MySQL应用参数的三种方式
- `set [session] 参数名=参数值  #设置当前会话（连接）参数`
- `set global 参数名=参数值  #设置全局参数`
- `设置应用配置文件`
	- `windows`存放在应用程序根目录中的`my.ini`
	- `linux`保存在`/etc/my.cnf`

# Connection连接参数
- `max_connections`：代表数据库同时允许的最大允许连接数
	- 连接有两种状态:`sleep`、`query`
		- `sleep`代表连接处于闲置状态
		- `query`代表连接正处于处理任务状态
		- `sleep`+`query`连接的总量不能超过`max_connections`的设置值
- `show status like 'Thread%'`
	- `Thread_cache`：共缓存过多少个连接，如果我们在MySQL服务器配置文件中设置了`Thread_cache_size`，当客户端断开之后，服务器处理次客户的线程将会缓存起来以响应下一个客户而不是销毁
	- `Thread_connected`:当前有多少个被打开的连接
	- `Thread_created`：历史总共创建过多少个数据库连接
	- `Thread_running`：当前正在执行的
- `show status like 'Max_used_connections%'`：显示历史最大连接数以及出现时间
- `back_log`：设置保存多少数据库请求到堆栈中，即连接数达到`max_connections`时，新来的请求将会被放在堆栈中，该堆栈的数量即`back_log`，如果等待连接的数量超过`back_log`，将会报`unauthenticated user |xxx.xxx.xxx|NULL|Connect....`
- `wait_timeout`:针对非交互式连接，如JDBC连接数据库，超过一定时间后，数据库自动关闭，默认8小时
- `interactive_timeout`:针对交互式连接，如mysql客户端连接，超过该时间，数据库自动关闭，默认8小时
- `show processlist`:查看数据库连接情况列表

# 查询缓存参数设置
- mysql8取消了该功能。5.7默认关闭
- 查询缓存，对于同样的`select`语句，区分大小写，才会直接从缓冲区中读结果。
- `set global query_cache_size=102400000;`：设置查询缓存大小
	- `show status like 'Qcache%'`--查看缓存相关信息
		- `Qchche_free_memory`:`Query_Cache`中目前剩余的内存大小。通过这个参数我们可以较为准确的观察当前系统中的`Query_Cache`内存大小是否足够，是需要增加还是过多了
		- `Qcache_lowmen_prunes`:多少条`Query`因内存不足而被清除出`Query_Cache`。通过这两个参数我们可以了解到缓存是否足够，如果该值不断增长，说明碎片化严重
		- `Qcache_total_blocks`:当前`Query_Cache`中的`block`的数量
		- `Qchche_free_blocks`:缓存中相邻内存块的个数。如果该值显示过大，则说明`Query_Cache`中的内存碎片较多。
		- 查询缓存碎片率：`Qcache_free_blocks`/`Qcache_total_block*100%`
		- 如果查询缓存碎片率超过20%，可以利用`flush query cache`整理缓存碎片
		- `Qcache_hits`:表示有多少次命中缓存
		- `Qcache_inserts`:表示多少次未命中而插入，即新的查询在缓存中未找到，执行查询处理后把结果`insert`到缓存中。
		- `Qcache_queries_in_cache`:当前缓存中缓存的查询次数
		- `Qcache_not_cached`:未进入查询缓存的`select`个数。
- `query_cache_limit`:超过此大小的查询将不会被缓存
- `query_cache_type`:缓存类型，决定缓存什么样子的查询。
	- 0：OFF禁用
	- 1：ON缓存所有结果
	- 2：DENAND，只缓存`select`语句中通过`SQL_CACHE`指定需要缓存的查询
- `query_cache_min_res_unit`:缓存块的最小大小。设置值较大对大数据查询有好处，但是都是小查询，则容易造成内存碎片和浪费。
- 每个查询占用缓存的平均值：`(query_cache_size-Qcache_free_memory)/Qcache_queries_in_cache`
- 查询缓存利用率：`(query_cache_size-Qcache_free_memory)/query_cache_size*100%`，25%以下则说明`query_cache_size`设置过大，80%以上且`Qcache_lowmem_prunes`>50说明过小或碎片太多
- 查询缓存命中率`Qcache_hits/(Qcache_hits+Qcache_inserts)*100%`
- 排序缓冲`sort_buffer_size`，每个需要排序的线程分配该大小的一个缓冲区，增加该值加速`order by`或`group by`。[[16.order by是怎么工作的]]

# InnoDB引擎参数设置
- `innodb_buffer_pool_size`: 专用于优化`innodb`引擎的。默认为128M。
- 查看一些参数
	- `innodb_buffer_pool_pages_data`:已使用的缓存页数量
	- `innodb_buffer_pool_pages_total`:全部缓存页数量
	- `innodb_page_size`每页长度
- 页面使用率`innodb_buffer_pool_pages_data/innodb_buffer_pool_pages_total`
	- >95%,考虑增大`innodb_buffer_pool_size`,建议使用物理内存的75%
- `innodb_flush_log_at_trx_commit`：控制日志合适何时硬盘 [[redo log与binlog]]
- `innodb_doublewrite`双写操作[[两次写]]
- `innodb_file_per_table`=1：设置独立的表空间文件xxx.ibd
- `innodb_thread_concurrency`:设置`innodb`线程的并发数，默认值是0表示不被限制。若要设置则与服务器的CPU核心数相同或是CPU核心数的两倍

# Centos7参数调优
- `/etc/sysctl.conf`配置文件
```
net.core.somaxconn=65535                #系统允许同时发起的TCP连接数，默认是128
net.core.netdev_max_backlog=65535       #缓冲区，当网络接口接收的速度大于内核处理的速度时，允许送到队列的数据包的最大数目
net.ipv4.tcp_max_syn_backlog=65535      #允许的半连接同步包上限
net.ipv4.tcp_fin_timeout=10             #用于设置
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_tw_recycle=1

net.core.wmem_default=87380
net.core.wmem_max=16777216
net.core.rmem_default=87380
net.core.rmem_max=16777216

net.ipv4.tcp_keepalive_time=100
net.ipv4.tcp_keepalive_intvl=10
net.ipv4.tcp_keepalive_probes=3

kernel.shmmax=2147483648
vm.swappiness=0
```

# MySQL服务器硬件选择
- CPU选择
	- 一定要选择64位的CPU与64位系统
	- 对于并发比较高的场景CPU数量比频率重要
	- CPU密集型场景和复杂SQL则频率越高越好
- 内存的选择
	- 理想选择：服务器内存大于数据总量。。。有点不可能
	- 内存频率越高处理速度越快
	- 内存总量小要合理组织热点数据，保证内存覆盖
	- 内存对写操作也有重要的性能影响
- 硬盘的选择
	- SSD
	- 混合硬盘
	- 机械硬盘

