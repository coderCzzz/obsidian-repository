## 数据页概述
### 组成
![[截屏2021-11-18 下午4.26.16.png]]

- [[#File Header文件头]]
- [[#Page Header页头]]


### File Header文件头
- 固定大小38字节
- 记录页的信息
- ![[截屏2021-11-18 下午4.27.48.png]]
- 看一下几个参数
	- `FIL_PAGE_OFFSET`：表示该页在所有页中的位置
	- `FIL_PAGE_PREV`和`FIL_PAGE_NEXT`：当前页的上一个页、下一个页
	- `FIL_PAGE_TYPE`：页类型，`0x45BF`代表数据页
	- `FIL_PAGE_LSN`:该页最后被修改的日志序列位置
	- `FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID`，页属于哪个表空间

### Page Header页头
- 固定大小，56字节
- 记录数据页的状态信息
- ![[截屏2021-11-18 下午4.32.01.png]]

### Infimum和Supermum Records
- 每个数据页有两个虚拟行记录，限定记录的边界
- `infimum`比该页中任何主键值都要小的值
- `Supermum`比该页中任何主键值都要大的值
### User Records行记录
- 就是行数据
### Free Space空闲空间
- 空闲列表
### Page Directory页目录
- B+树索引本身并不能找到具体的一条记录，能找到是该记录所在的页
- 数据库把页载入到内存，再通过`Page Directory`进行二叉查找。
### File Trailer文件结尾信息
- 固定大小8字节
- 为了检测页是否已经完整地写入磁盘。