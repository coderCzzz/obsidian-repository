## change buffer
### 作用
更新一个数据页时，如果该数据页在内存就直接更新。如果不在，就先把这个修改操作记录的`change buffer`中，等下一次访问对应数据页时，把相应数据页读入内存，执行`change buffer`中与这个页有关的操作。
减少了读磁盘的损耗，且数据读入内存需要消耗`buffer pool`，因此还能够避免占用内存，提高内存利用率。
### merge时机
merge：将`change buffer`中的操作应用到原数据页，得到最新结果的操作称为`merge`。
- 访问对应的数据页触发`merge`
- 系统后台线程定期`merge`
- 数据库正常关闭，`merge`

### 何时会使用change buffer
- 更新唯一索引时，所有的操作都需要检测是否违反唯一性约束，因此必须将数据页读入内存页。故此时没必要使用`change buffer`
- 只有普通索引可以用。
- 什么场景可以用
    - 对于写多读少的业务，效果最好。写少读多的业务，写完后立即查询，磁盘IO没有增加，反而增加了维护`change buffer`的成本

### 设置change buffer
- `change buffer`使用的是`buffer pool`的内存。
- 可以通过`innodb_change_buffer_max_size`动态设置，数值表示占`buffer pool`的百分比


## buffer pool