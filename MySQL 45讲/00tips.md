## InnoDB行锁是通过锁索引实现的 ^1
> 如果update的列没有索引,即使只update一条记录,也会锁定整张表

## 立即启动一个事务 ^2

> begin/start transaction命令不会立刻启动一个事务;只有执行后的第一个操作InnoDB表语句(第一个快照读语句)事务才会启动;
>
> `start transaction with consisten snapshot`语句执行后会立刻启动一个事务
> 该语句的含义是创建一个持续整个事务的一致性快照,因此只有RR级别下才有效.对于RC级别由于每个执行都会起一个事务,就没有意义了.[参考](08讲事务到底是隔离还是不隔离的.md#RR和RC是如何通过一致性读实现的)

## change_buffer ^3
> 当更新一个数据页是,如果数据页在内存中就直接更新.如果不在内存中InnoDB会将这些更新操作缓存在change buffer中,这样就不需要从磁盘读入数据页了.
> 
> 下次再查询需要访问该数据页时,吧数据页读入内存然后执行change buffer中与这个页相关操作.这样就保证了逻辑性
> 他是持久化的,会被异步刷入到磁盘中
> 
> 将change buffer中的数据操作应用到原始也,称为merge;在访问数据页时触发,系统有后台线程会定期merge.数据库正常关闭过程中也会merge
> change_buffer使用的是buffer pool的内存,因此不是无限增大的.可以通过`innodb_change_buffer_max_size`来动态设置.这个参数设置为50的时候表示change_buffer大小最多只能占用buffer_pool的50%.
> 
> 对于写多读少的业务来说,页面在写完后马上被访问到的概率比较小.此时change buffer使用效果最好.如果是写完后马上查询的场景,那就没必要使用了(需要维护merge)

## 慢查日志 ^4
> 记录MySQL中响应时间超过阈值的语句,即运行时间超过`long_query_time`值的SQL就会被记录
> 默认慢查日志不会打开可以通过`slow_query_log`设置打开

### 慢查日志相关参数
- `slow_query_log` 1表示开启慢查日志，0 表示关闭。
-` log-slow-queries` 旧版（5.6 以下版本）慢查询日志存储路径默认给一个缺省的文件 host_name-slow.log
- `slow-query-log-file` **新版（5.6 及以上版本）** 慢查询日志存储路径;默认给一个缺省的文件 host_name-slow.log
- `long_query_time` 慢查询阈值，当查询时间多于设定的阈值时，记录日志。
- `log_queries_not_using_indexes` 未使用索引的查询也被记录到慢查询日志中（可选项）。
- `log_output` 日志存储方式
	- log_output='FILE'表示将日志存入文件，默认值是'FILE'。
	- log_output='TABLE'表示将日志存入数据库
		- 这样日志信息就会被写入到 mysql.slow_log 表中
		- 相比文件更加占用资源
	- MySQL 数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output='FILE,TABLE'

## 脏页
> 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘 后，内存和磁盘上的数据页的内容就一致了，称为“干净页”.

## TPS
指的是事务数/秒 Transactions Per Second;用来衡量数据库的处理能力