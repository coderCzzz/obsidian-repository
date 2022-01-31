# 慢SQL日志
## 配置
- 开启慢SQL日志`set GLOBAL slow_query_log=on`
- 慢SQL执行时间阈值`set global long_query_time=0.001`单位秒
- 慢SQL日志保存位置`set global query_log_file='slow-sql.log'`
- `set global log_queries_not_using_indexs=on`记录没有使用索引的SQL（系统表执行的也会继续下来）
- 上面设置重启会失效，可以在`my.cnf`配置中修改

## 慢SQL日志内容
- 执行时间Time
- 环境
- 查询时间Query_time
- 资源锁定时间Lock_time
- 查询结果总行数Rows_sent
- 扫描的行数Rows_examined
- 时间戳
- 最后是SQL原文

# Explain执行计划
```
explain select (select 1 from actor where id=1)
	from
	(select * from film where id=1
	union
	select * from film where id=1
	) der;
```
- explain 对应的sql
- 结果中字段的含义
	- `id`:执行计划的编号，`select`的序列号，有几个`select`就有几个id，并且id的顺序是按`select`出现的顺序增长的。多表时，id=1是驱动表
	- `select type`查询类型
		- `simple`简单查询，不包含子查询和`union`
		- `PRIMARY`复杂查询中最外层的查询
		- `DERIVED`:包含在from子句中的子查询。mysql会将结果存放在一个临时表，也称为派生表。
		- `UNION`：union关键字之后的查询
		- `UNION RESULT` ：union连接后的去重操作,也就是从union临时表检索结果的select
		- `SUBQUERY`：select中的子查询
	- `table`：访问的表
	- `partitions`:分区表
	- `type`:表示关联类型或访问类型，即`MySQL`决定如何查找表中的行
		- 根据执行效率从高到低如下：
			- `system`
			- `const`:常量引用，对查询的某部分优化将其转化成一个常量，用于主键或唯一键
			- `eq_ref`：关联查询，一张表的主键和另一张表的外键关联查询
			- `ref`：非主键或非唯一键的数据检索，不使用唯一索引
			- `fulltext`：全文索引，极少使用
			- `ref_or_null`：类似ref，可以搜索值为null的行
			- `index_merge`
			- `unique_subquery`
			- `index_subquery`
			- `range`：范围检索
			- `index`：基于索引进行全表检索
			- `All`：纯粹全表检索
	- `possible_key`:显示查询可能使用哪些索引来查找
	- `key`:最终使用的索引
	- `key_len`:显示mysql在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列
	- `ref`:显示关联查询引用的列
	- `rows`:mysql估计要读取并检测的行数
	- `Extra`:显示额外信息
		- `distinct`:一旦mysql找到了与行相联合匹配的行，就不再搜索了
		- `using index`:返回的列数据只使用了索引中的信息，而没有再去访问表中的行记录
		- `Using where`:在存储引擎检索行后再进行过滤
		- `Using temporary`:使用临时表计算排序，效率很差
		- `Using filesort`:采用文件扫描对结果进行