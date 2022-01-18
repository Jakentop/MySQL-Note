# 日志系统
> innodb事务日志包括redo log和[undo log](08讲事务到底是隔离还是不隔离的.md#^137642).redo log是重做日志,提供前滚操作(MVCC的物理实现,以及备份).undo log是回滚日志,提供回滚操作.
## redo log 日志模块(InnoDB 专有)
### 概念

#### redolog有两个部分:

- 日志缓冲(redo log buffer)
- 磁盘上重做日志文件(redo log file)持久的

### 处理逻辑

- 当一条记录更新的时候,InnoDB会先把记录写到redo log中,并且更新内存
	- 通过WAL技术,在适合的时候再把redo log更新到磁盘中
	- WAL(Write-Ahead Loggin)先写内存,在写磁盘
- 数据在内存中刷新后,会在空闲的时候(或临时内存已满)时刷新脏页到磁盘持久化

### redo log写入方式
- redo log是固定大小的,由配置来设置大小
- 如果redo log写满了就会继续从头开始写:
- checkpoint:需要擦除的位置(环的末尾)
	- checkPoint前的内容都到write pos后内容是可以写入的
	- checkPoint后的内容到write pos前的内容都是没有刷新的内容
- write pos:当前记录的位置

![](http://img.jaken.top/image/202111241029099.png)
![](http://img.jaken.top/image/20211206192737.png)


### redo log的刷新方式

#### force log at commit持久化事务

- 事务提交是,先将事务的所有事务日志写到磁盘的redo log file和undo log file中持久化
- 默认每次写入都会调用fsync()系统调用(将OS buffer刷新到磁盘中)
- 支持用户自定义commit是如何吧log buffer的日志刷log file中
  - 通过`innodb_flush_log_at_trx_cimmit`的0,1,2;默认为1
  - 0:事务提交时不会写入,而是每秒调用fsync()写入;会丢失一秒内发生的数据
  - 1:事务提交是都会把log buffer刷新,但是IO性能较差;不会丢失数据
  - 2:每次提交都写入os buffer,然后每秒调用fsync()刷入磁盘;会丢失数据
- 主从模式中,要保证事务和持久性一致可以设置:
  - `sync_binglog=1`启动binlog后,每次事务同步写入磁盘
  - `innodb_flush_log_at_trx_commit=1`每次提交事务都写入磁盘

### log block 日志块

- innodb中redo log是以块为单位存储的,每个块占512字节
- 每个块由日志块头,日志块尾和日志主体组成
  - 日志块头 12字节
  - 日志块尾 8字节
  - 日志主体 492字节
- 当一个数据页产生的变化需要超过492字节来记录,就会用多个redo log block
- 日志块头包括:
  - log_block_hdr_no 4Bit 在redo log buffer中的位置ID
  - log_block_hdr_data_len 2Bit 改快中已记录的大小,写满时为0x200即512字节
  - log_block_first_rec_group 2Bit 第一个log的开始偏移位置
    - 即一个日志块除去块头可能并不是第一个log的开始位置
    - 例如一个日志超出492字节,就需要两个日志块来记录
  - log_block)checkpoint_no 4Bit 写入检查点信息的位置
- redo log buffer 或 redo log file on disk就是由很多log block组成的

### crash-safe

> 通过redo log即使数据库突然挂了,之前提交的记录(没有刷新的脏页)依旧可以通过redo log恢复.

## binlog 日志模块(MySQL的日志)
> Server层有自己的日志,称为binlog(归档日志),binlog没有[crash-safe](#crash-safe),所以InnoDB实现时使用了另一套日志系统实现此功能

### 不同点
| redo log                                                     | binlog                       |
| ------------------------------------------------------------ | ---------------------------- |
| InnoDB引擎特有                                               | MySQL Server层实现           |
| 物理日志:记录的数据页发生什么修改                            | 逻辑日志:记录某行语句的含义  |
| 循环写,总空间是固定的                                        | 不会覆盖原有日志,分文件      |
| 在binlog后写入(两阶段提交)                                   | 在redo log提交前,被提交      |
| 在数据修改前就会写入缓存中的redo log                         | 实体提交后一次性写入缓存日志 |
| 一个事务可能有多次记录<br/>最后一个提交的事务会覆盖所有未提交的事务记录 | 日志记录顺序和提交顺序有关   |

### 两种记录模式
- statement模式:记录的是SQL语句
- row模式:记录的是行内容(记录更新前内容和更新后内容),也是SQL语句的形式记录的

### 更新操作流程
- 执行器通过引擎查找到需要更新的行;如果该行所在页不在内存,则读入内存中
- 执行器拿到数据后,更新数据,并通过引擎写入接口写入新数据
- 引擎吧这行更新到内存中,同时写入redo log.此时redo log处于prepare状态.随后返回提交事务
- 执行器生成操作的binlog,并写入磁盘
- 执行器调用引擎的事务提交接口,引擎吧redo log改为提交状态(commit)

#### 执行流程图
![](http://img.jaken.top/image/202111241735320.png)

### 两阶段提交
> 即redo log回先处于prepare状态,在bin log提交后,redo log再提交变成commit状态

#### 如何恢复数据库
- bin log会记录所有的逻辑操作,并且采用追加写的形式
- 备份系统会对bin log做热备份或者冷备份,取决于数据重要性
- 如果需要恢复到某一时刻,则首先找最近的一次全量备份
- 然后从备份时间器,把bin log重放到误删前的时刻

#### 采用两阶段提交原因
> 由于两个log的提交不可能同时发生,因此他们的先后提交时间会有一定的影响
> 当在主从环境下,从库的同步就是通过binlog来实现的,所以这种情况很容易发送

##### 先写redo log再写bin log
- 假设redo写完了,bin没写就挂了.
- 系统重启后,会把redo log中的数据恢复[crash-safe](#crash-safe)
- bin log中没有记录,这就导致出现不一致性了

##### 先写bin log再写redo log
- bin log 写完后挂了
- redo log由于没有写,所以这个事务被回滚了
- 这就导致出现了不一致性

##### 两阶段提交 
- redo log 写完(prepare),然后挂了
- 重启后发现redo log是prepare所以这个事务没有提交,直接被回滚了



