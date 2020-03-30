# MySQL的Undo-log与Redo-log的实现机制

​	本文将详细介绍什么是`Undo Log`、`Redo Log`。

​	MySQL默认的Innodb在**REPEATABLE READ**隔离级别下,是如何通过`MVCC` + `Undo Log`，解决幻读的。

# 1. Undo Log是什么

## 1.1 简介

`Undo Log` ：
​	`undo`意为取消，以撤销操作为目的，返回指定某个状态的操作
​	`undo log`指事务开始之前，在操作任何数据之前,首先将需操作的数据备份到一个地方 (`Undo Log`)

**UndoLog是为了实现事务的原子性而出现的产物**

`Undo Log`实现事务原子性：
​	事务处理过程中如果出现了错误或者用户执行了 `ROLLBACK`语句,MySQL可以利用`Undo Log`中的备份将数据恢复到事务开始之前的状态

`Undo Log`**在MySQL Innodb存储引擎中用来实现多版本并发控制**

`Undo log`实现多版本并发控制：
​	事务未提交之前，Undo保存了未提交之前的版本数据，Undo 中的数据可作为数据旧版本快照供其他并发事务进行快照读

------

## 1.1 案例

如下图

​	事务A 执行`update`之前，备份数旧数据 --> Undo buffer -->落地 Undo Log 

​	事务B 此时查询的是Undo buffer中的内容（相当于读取的是快照，实现多版本并发控制）

​	事务A 若因意外rollback，会从Undo buffer中数据恢复（实现事务原子性）

![121120042327141](http://ww3.sinaimg.cn/large/006tNc79gy1g5xaswwkb5j30t50fadjt.jpg)

## 1.2 当前读、快照读

**快照读**：

​	SQL读取的数据是快照版本，也就是历史版本，普通的SELECT就是快照读Innodb快照读，数据的读取将由 cache(原本数据) + undo(事务修改过的数据) 两部分组成

**当前读**：

​	SQL读取的数据是最新版本。通过锁机制来保证读取的数据无法通过其他事务进行修改

​	`UPDATE`、`DELETE`、`INSERT`、`SELECT … LOCK IN SHARE MODE`、`SELECT … FOR UPDATE`都是当前读

# 2. Redo Log是什么

## 2.1 简介

`Redo Log`：
​	`Redo`，顾名思义就是重做。以恢复操作为目的，重现操作；
​	`Redo log`指事务中操作的任何数据,将最新的数据备份到一个地方 (`Redo Log`)

Redo log的持久：
​	不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入redo 中。具体的落盘策略可以进行配置

**RedoLog是为了实现事务的持久性而出现的产物**

`Redo Log`实现事务持久性：
​	防止在发生故障的时间点，尚有脏页未写入磁盘，在重启MySQL服务的时候，根据`Redo Log`进行重做，从而达到事务的未入磁盘数据进行持久化这一特性。

------

![121120042327151](http://ww4.sinaimg.cn/large/006tNc79gy1g5xat1l7hfj30kz0iu42z.jpg)

## 2.2 Redo Log的落盘配置

​	指定Redo log 记录在{datadir}/ib_logfile1&ib_logfile2 可通过`innodb_log_group_home_dir` 配置指定目录存储

```mysql
mysql> show variables like "innodb_log_group_home_dir";
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| innodb_log_group_home_dir | ./    |
+---------------------------+-------+
1 row in set (0.01 sec)

mysql>
```

一旦事务成功提交且数据持久化落盘之后，此时Redo log中的对应事务数据记录就失去了意义，所以Redo log的写入是日志文件循环写入的

指定Redo log日志文件组中的数量 `innodb_log_files_in_group` 默认为2

指定Redo log每一个日志文件最大存储量`innodb_log_file_size` 默认48M

指定Redo log在cache/buffer中的buffer池大小`innodb_log_buffer_size` 默认16M

Redo buffer 持久化Redo log的策略`Innodb_flush_log_at_trx_commit`：

```mysql
mysql> show variables like "Innodb_flush_log_at_trx_commit";
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 0     |
+--------------------------------+-------+
1 row in set (0.01 sec)

mysql> 
```

- 取值 **0** 每秒提交 Redo buffer --> Redo log OS cache -->flush cache to disk[可能丢失一秒内的事务数据]
- 取值 **1** 默认值，每次事务提交执行Redo buffer --> Redo log OS cache --> flush cache to disk[最安全，性能最差的方式]
- 取值 **2** 每次事务提交执行Redo buffer --> Redo log OS cache 再每一秒执行 --> flush cache to disk操作