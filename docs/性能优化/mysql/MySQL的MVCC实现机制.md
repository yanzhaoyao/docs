# MySQL的MVCC实现机制

本文将详细介绍MySQL的`MVCC`实现机制，`MVCC`能解决什么问题

# 前言

先思考一个问题

```mysql
## 伪代码
## 查看mysql的设置的事务隔离级别
select @@tx_isolation;
ex1：
	tx1: set session autocommit=off;
		 update users set lastUpdate=now() where id =1;
		 ## 在未做commit/rollback操作之前
		 ## 在其他的事务我们能不能进行对应数据的查询（特别是加上了X锁的数据）
	tx2: select * from users where id > 1;
			 select * from users where id = 1;
ex2：
	tx1: begin
		 select * from users where id =1 ;
	tx2: begin
		 update users set lastUpdate=now() where id =1;
	tx1:
		 select * from users where id =1;
```

这两个案例从结果上来看是一致的！底层实现是怎样的呢？是一样的吗？他们的底层实现跟MVCC有什么关系么？

## 1. MVCC是什么？

MVCC：
​	Multiversion concurrency control (多版本并发控制)

普通话解释：
​	并发访问(读或写)数据库时，对正在事务内处理的数据做多版本的管理。以达到用来避免写操作的堵塞，从而引发读操作的并发问题。

## 2. MVCC实现

​	MVCC是通过保存数据在某个时间点的快照来实现的. 不同存储引擎的MVCC. 实现是不同的,典型的有乐观并发控制和悲观并发控制.

## 3. MVCC的具体实现分析

下面,我们通过InnoDB的MVCC实现来分析MVCC是怎样进行并发控制的

InnoDB的MVCC,是通过在每行记录后面保存两个隐藏的列来实现的,这两个列，分别保存了这个行的创建时间（`DB_TRX_ID`），一个保存的是行的删除时间（DB_ROLL_PT）。这里存储的并不是实际的时间值,而是系统版本号(可以理解为事务的ID)，每开始一个新的事务，系统版本号就会自动递增，事务开始时刻的系统版本号会作为事务的ID.下面看一下在**REPEATABLE READ**隔离级别下,MVCC具体是如何操作的.

### 3.1 MySQL中MVCC逻辑流程

#### 3.1.1 插入

假设系统的全局事务ID号从1开始；

```mysql
begin;  -- 拿到系统的事务ID=1;
insert into teacher(name,age) value ('sever',18);
insert into teacher(name,age) value ('qingshan',19);
commit;
```

如下图，数据插入成功后，表后面两列保存相应的版本号

![121120042327111](http://ww3.sinaimg.cn/large/006tNc79gy1g5xatevo8kj31at0p8q8s.jpg)

#### 3.1.2 删除

假设系统的全局事务ID号目前到了22

```mysql
begin; -- 拿到系统的事务ID=22;
delete teacher where id = 1;
commit;
```

如下图，id为2的数据行，删除版本号设置为当前事务ID(22)

![121120042327115](http://ww2.sinaimg.cn/large/006tNc79gy1g5xath2aqkj30yo0nuwit.jpg)

#### 3.1.3 修改

假设系统的全局事务ID号目前到了33

```mysql
begin; -- 拿到系统的事务ID=33;
update teacher set age = 19 where id = 1;
commit;
```

修改操作是先做命中的数据行的copy，将原行数据的删除版本号的值设置为当前事务ID(33)

![121120042327119](http://ww3.sinaimg.cn/large/006tNc79gy1g5xatk0vcgj317u0ouah3.jpg)



#### 3.1.4 查询

数据行查询规则

1. 查找数据行版本早于当前事务版本的数据行（也就是，行的系统版本号小于或等于事务的系统版本号），这样可以确保事务读取的行，要么是在事务开始前已经存在的，要么是事务自身插入或者修改过的
2. 查找删除版本号要么为null，要么大于当前事务版本号的记录，确保取出来的行记录在事务开启之前没有被删除

只有1,2同时满足的记录，才能返回作为查询结果



假设系统的全局事务ID号目前到了44

```mysql
begin; -- 拿到系统的事务ID=44;
select * from teacher;
commit;
```

![121120042327123](http://ww2.sinaimg.cn/large/006tNc79gy1g5xatn74m3j319k0rrn49.jpg)

### 3.2 MySQL中版本控制案例

数据准备：

```mysql
insert into teacher(name,age) value ('seven',18) ;
insert into teacher(name,age) value ('qingshan',20) ;
# tx1:
begin; 									-- --------1
select * from users ; 					-- --------2
commit;
# tx2：
begin; 									-- --------3
update teacher set age =28 where id =1; -- --------4
commit;
```

案例1
按顺序执行 1，2，3，4，2

案例2
按顺序执行 3，4，1，2

#### 3.2.1 案例一（1，2，3，4，2）

tx1 先执行1,2

tx2 再执行3,4

tx1 再执行2

![121120042327130](http://ww2.sinaimg.cn/large/006tNc79gy1g5xatr1ii0j30zi0nnjv8.jpg)



#### 3.2.2 案例二（3、4、1、2）

tx2 先执行3，4

tx1 再执行1，2

![121120042327134](http://ww4.sinaimg.cn/large/006tNc79gy1g5xattyk43j30y10my434.jpg)

案例二查询结果不是我们想要的，mysql的Innodb也不是这样做的。



请看下一篇MySQL中`Undo log`和`Redo log`的讲解