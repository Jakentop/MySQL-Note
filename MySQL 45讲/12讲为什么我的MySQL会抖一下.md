> 有些时候SQL执行会突然抖动,正常执行很快,但是突然就特别慢,而且很难复现这种场景

## 为什么SQL语句变慢了
- 偶尔抖一下有可能是在刷新内存中的[脏页](00tips.md#脏页)
- 在以下情况会出现脏页的刷新
	- redo log写满了,此时系统会停止所有更新操作,吧 [02讲日志系统;一条SQL更新语句是如何执行的](02讲日志系统;一条SQL更新语句是如何执行的.md) 往前推(刷新到磁盘中)
	- 当系统内存不足是,需要新的内存也,则需要淘汰一些脏页(刷新他们到磁盘)
		- 虽然可以直接淘汰内存,并记录下次访问脏页时,从redo log中恢复,但是这种方案性能过差
		- 数据页的两种状态:
			- 内存中存在,内存中的数据页可能是脏页,但是一定是最新的
			- 内存中没有,他一定是干净页,但是访问他需要先把他读入到内存中
	- 在系统触发刷新redolog的时候新的请求进来了
	- 当MySQL关闭时,需要刷新内存中所有的脏页

## InnoDB用缓冲池管理内存
- 缓冲池中内存页也有三种状态
	- 没有使用的的内存
	- 使用了,并且存放的是干净页
	- 使用了,并且存放的是脏页
- InnoDB的策略是尽量使用内存,因此对于一个长时间运行的库来说，未被使用的页面很少
- 读入数据页不在内存时,会到缓冲池申请数据页,如果此时缓冲池满,则需要用LRU淘汰
    - 如果淘汰是干净页则不需要刷新磁盘
    - 如果淘汰是脏页则需要先刷新磁盘
- 有两种情况会**明显影响性能**:
    - 一个查询要淘汰很多脏页导致响应时间变长
    - redoLog写满,此时数据库直接无法提供服务,每部刷新

## InnoDB刷脏页的控制策略

### 设置IOPS磁盘能力
`innodb_io_capacity`此参数告诉InnoDB磁盘能力.建议设置为磁盘IOPS;
如果**遇到InnoDB的[TPS](00tips.md#TPS)很低,同时磁盘IO也很低,可能是因为此参数设置过小了**

### 脏页计算方式
- InnoDB的刷盘速度就是要参考这两个因素：**脏页比例,redo log写盘速度**.
- `innodb_max_dirty_pages_pct`是脏页比例上限,默认为75%.即超过这个比例就会强制刷新
    - 设M为当前脏页比例
    - 当前脏页数(0-100)为F1(M)
    - F1(M):M>=innodb_max_dirty_pages_pct?100:100\*M/innodb_max_dirty_pages_pct
- InnoDB每次写入日志都有一个序好
    - 设N:当前序号和checkpoint对应的序号差
    - F2(N)为一个0-100的值,N越大结果越大(计算过程复杂省略)
- **算得的F1(M)和F2(N)两个值，取其中较大的值记为R，之后引擎就可以按 照innodb_io_capacity定义的能力乘以R%来控制刷脏页的速度**

#### 图示
![](http://img.jaken.top/image/202201181056351.png)

#### 总结
- 平时要多关注脏页比例，不要 让它经常接近75%
- 脏页比例:Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total
- 计算sql:
MySQL 5.7.6以前版本
```sql
# 脏页数量
select VARIABLE_VALUE into @a from global_satus where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty'; 
# 总页数量
select VARIABLE_VALUE into @b from global_satus where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'; 
select @a/@b;
```
MySQL 5.7.6以后版本
``` sql
# 总脏页数  
select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';  
# 总页数  
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';  
select @a/@b;
# 可以使用show指令
show global status like 'Innodb_buffer_pool_pages_dirty';
```

### 邻近刷新策略`innodb_flush_neighbors`
- 当刷新某页时会对相邻页数个页做刷新
- 刷新数量由`innodb_flush_neighbors`来决定
- 对于机械硬盘很有意义(因为随机IO太小)
- 但是对于SSD来说,邻近刷新毫无意义
- MySQL 8.0后该参数默认值为0