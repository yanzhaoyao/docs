# MySQL锁

​	MySQL为什么要提供锁机制？锁能解决什么问题？

​	如何保证数据并发访问的一致性、有效性是所在有数据库必须解决的一个问题，锁冲突也是影响数据库并发访问性能的一个重要因素。从这个角度来说，锁对数据库而言显得尤其重要，也更加复杂。

# 理解表锁、行锁

锁是用于管理不同事务对共享资源的并发访问

表锁与行锁的区别：

锁定粒度：表锁 > 行锁

加锁效率：表锁 > 行锁

冲突概率：表锁 > 行锁

并发性能：表锁 < 行锁

InnoDB存储引擎支持行锁和表锁（另类的行锁）



# MySQL Innodb锁类型

## 3种类型

**行锁**

​	共享锁（行锁）：Shared Locks

​	排它锁（行锁）：Exclusive Locks

**表锁**

​	意向锁共享锁（表锁）：Intention Shared Locks

​	意向锁排它锁（表锁）：Intention Exclusive Locks

**自增锁**

​	AUTO-INC Locks

------

下面3种是**行锁的算法**

**记录锁** Record Locks

**间隙锁** Gap Locks

**临键锁** Next-key Locks

行锁的算法https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html

# 共享锁（Share Locks）vs 排它锁（Exclusive Locks）

**共享锁**:又称为读锁，简称**S**锁，顾名思义，共享锁就是多个事务对于同一数据可以共享一把锁，都能访问到数据，但是只能读不能修改;

加锁释锁方式：

```mysql
select * from users WHERE id=1 LOCK IN SHARE MODE;
commit/rollback
```

实例测试Share Locks

会话A autocommit关闭

![1544451397852](http://ww4.sinaimg.cn/large/006tNc79gy1g5xakoedvpj30m20ba0t3.jpg)

```mysql
##1、当前会话A autocommit关闭
mysql> set session autocommit = OFF;
Query OK, 0 rows affected (0.00 sec)

mysql> show VARIABLES like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.01 sec)

##2、查询select
mysql> select * from users WHERE id=1 LOCK IN SHARE MODE\G
*************************** 1. row ***************************
        id: 1
     uname: 李二狗
 userLevel: 2
       age: 19
  phoneNum: 13666666666
createTime: 2018-12-01 15:39:46
lastUpdate: 2018-12-01 15:39:50
1 row in set (0.00 sec)
##3、当前事务还没提交或者回滚
```

会话B autocommit采用默认的，未关闭

![1544451620296](http://ww4.sinaimg.cn/large/006tNc79gy1g5xamhdzyoj30jn0b7dg6.jpg)

```mysql
mysql> select * from users where id =1\G
*************************** 1. row ***************************
        id: 1
     uname: 李二狗
 userLevel: 2
       age: 19
  phoneNum: 13666666666
createTime: 2018-12-01 15:39:46
lastUpdate: 2018-12-01 15:39:50
1 row in set (0.00 sec)

mysql> update users set age=19 where id =1;
##这里修改会阻塞。。。
##。。。等一会显示
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

![1544451692396](http://ww3.sinaimg.cn/large/006tNc79gy1g5xan0v2zaj30ex063dfv.jpg)

**排他锁**:又称为写锁，简称**X**锁，排他锁不能与其他锁并存，如一个事务获取了一个数据行的排他锁，其他事务就不能再获取该行的锁（共享锁、排他锁），只有该获取了排他锁的事务是可以对数据行进行读取和修改，（其他事务要读取数据可来自于快照//TODO 快照后面会将，待补充链接）

加锁释锁方式：
`delete` / `update` / `insert` 默认加上X锁

```mysql
SELECT * FROM table_name WHERE ... FOR UPDATE
commit / rollback 
```

实例测试

会话A

![1544452646053](http://ww4.sinaimg.cn/large/006tNc79gy1g5xanjtg6wj30m10bcjrp.jpg)

```mysql
mysql> set session autocommit = OFF;
Query OK, 0 rows affected (0.01 sec)

mysql> show VARIABLES like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.01 sec)

mysql> select * from users where id =1 for update\G
*************************** 1. row ***************************
        id: 1
     uname: 李二狗
 userLevel: 2
       age: 19
  phoneNum: 13666666666
createTime: 2018-12-01 15:39:46
lastUpdate: 2018-12-01 15:39:50
1 row in set (29.50 sec)
##此时会话A拿到排它锁
```

会话B

```mysql
mysql> show VARIABLES like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.02 sec)

mysql> select * from users where id =1 for update\G
## 会阻塞。。。
## 然后过一段时间超时
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

##在尝试拿共享锁（会话A要重新拿一次排它锁，因为我用的linux，这边超时了，会话A的事务无效了）
mysql> select * from users where id =1 lock in share mode\G
## 会阻塞。。。
## 然后过一段时间超时
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

![1544453204545](http://ww1.sinaimg.cn/large/006tNc79gy1g5xanwst0qj30fc01e3yc.jpg)

![1544453162814](http://ww2.sinaimg.cn/large/006tNc79gy1g5xao55olrj30f501e0sk.jpg)

## Innodb到底锁了什么？

InnoDB的行锁是通过给索引上的索引项加锁来实现的。

只有通过索引条件进行数据检索，InnoDB才使用行级锁，否则，InnoDB将使用**表锁（锁住索引的所有记录）**

表锁：lock tables xx read/write；

测试，换用Navicat客户端测试

看一下users表的DDL

`phoneNum`是一个普通的字段

`index_name` 这是一个单列索引（是一个冗余的索引，仅仅为了测试使用，请忽略冗余）

`idex_union_name&userLevel&age` 联合索引

![1544453549354](http://ww3.sinaimg.cn/large/006tNc79gy1g5xap0l9e4j30hx06jdg3.jpg)

ex1:

where条件在普通列上

![1544453493730](http://ww1.sinaimg.cn/large/006tNc79gy1g5xap4td9cj30g80aldfy.jpg)

ex1:此时执行where id=2;where id=1;发现都是阻塞

![1544454112388](../../images/optimize/mysql/1544454112388.png)

ex2:

where条件在主键上

![1544454240886](http://ww2.sinaimg.cn/large/006tNc79gy1g5xap9y6utj30c30ehjrj.jpg)

发现 where id = 2  正常执行

![1544454312774](http://ww4.sinaimg.cn/large/006tNc79gy1g5xapcrmbwj30c205o74d.jpg)



发现where id = 1 阻塞 超时了

![1544454459603](http://ww1.sinaimg.cn/large/006tNc79gy1g5xapeyrudj30cr05qdfx.jpg)

ex3:

where条件在索引上

seven 的行记录

![1544454887749](http://ww2.sinaimg.cn/large/006tNc79gy1g5xapheq4gj30jo0330sp.jpg)

![1544454544263](../../images/optimize/mysql/1544454544263.png)

where uname='sever',阻塞

![1544454642631](http://ww4.sinaimg.cn/large/006tNc79gy1g5xapks8r4j30g5062jrp.jpg)

where 条件id = 1,执行正常

![1544454798410](http://ww3.sinaimg.cn/large/006tNc79gy1g5xapnnfl3j30fb05lmxe.jpg)

where 条件在seven 的id =4 ，阻塞了

![1544455038483](http://ww3.sinaimg.cn/large/006tNc79gy1g5xaprc1jgj30fh05z3yt.jpg)

# 意向共享锁（IS）& 意向排他锁

**意向共享锁(IS)**

​	表示事务准备给数据行加入共享锁，即一个数据行加共享锁前必须先取得该表的IS锁，意向共享锁之间是可以相互兼容的

**意向排它锁(IX)**

​	表示事务准备给数据行加入排他锁，即一个数据行加排他锁前必须先取得该表的IX锁，意向排它锁之间是可以相互兼容的

**意向锁(IS、IX)**是InnoDB数据操作之前自动加的，不需要用户干预

意义：
当事务想去进行锁表时，先尝试拿意向锁，意向拿不到，就不用去拿共享锁、排他锁	

例如生活中的案例

​	一节火车车厢上的卫生间WC会有一个指示灯，提示**有人、无人**，其他乘客只需要通过指示灯可以判断卫生间能否进入，获取使用权。这个指示灯就相当于意向锁，只是一个标识。

​	乘客要获取使用权卫生间，不用进入卫生间查看是否有人，只需要看指示灯就行了。

​	事务要获取一个数据行的锁，要先获取意向锁。如果意向所获取不到，就没必要继续获取其共享锁或排他锁了，提高获取锁的性能。

# 自增锁 AUTO-INC Locks

针对自增列自增长的一个特殊的表级别锁

```mysql
mysql> show variables like 'innodb_autoinc_lock_mode';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_autoinc_lock_mode | 1     |
+--------------------------+-------+
1 row in set (0.00 sec)
```

默认取值1，步长为1，代表连续，事务未提交ID永久丢失

# 临键锁（Next-key）&间隙锁（Gap）&记录锁（Record）

`Gap locks`：

**锁住数据不存在的区间（左开右开）**
​	当sql执行按照索引进行数据的检索时，查询条件的数据不存在，这时SQL语句加上的锁即为Gap locks，锁住索引不存在的区间（左开右开）

`Record locks`：

**锁住具体的索引项**
​	当sql执行按照唯一性（Primary key、Unique key）索引进行数据的检索时，查询条件等值匹配且查询的数据是存在，这时SQL语句加上的锁即为记录锁Record locks，锁住具体的索引项

下面结合实例详细介绍

数据准备，比如数据库中有一个表t,表结构和数据如下：

```mysql
mysql> desc t;
+-------+---------+------+-----+---------+----------------+
| Field | Type    | Null | Key | Default | Extra          |
+-------+---------+------+-----+---------+----------------+
| id    | int(11) | NO   | PRI | NULL    | auto_increment |
| value | int(11) | NO   |     | NULL    |                |
+-------+---------+------+-----+---------+----------------+
2 rows in set (0.00 sec)

mysql> select * from t;
+----+-------+
| id | value |
+----+-------+
|  1 |     1 |
|  4 |     7 |
|  7 |     7 |
| 10 |    10 |
+----+-------+
4 rows in set (0.00 sec)
```

很简单的数据，只有4条

## 临键锁（Next-key）

`Next-key locks`：
​	**锁住记录+区间（左开右闭）**
​	当sql执行按照索引进行数据的检索时,查询条件为范围查找（between and、<、>等）并有数据命中则此时SQL语句加上的锁为Next-key locks，锁住索引的记录+区间（左开右闭）

![120910194139074](http://ww4.sinaimg.cn/large/006tNc79gy1g5xaqe86b6j30ve0andhc.jpg)



为什么Innodb选择临键锁Next-key作为行锁的默认算法？

## 间隙锁（Gap）

![120910194139079](http://ww3.sinaimg.cn/large/006tNc79gy1g5xaqfhvu4j30qu0bftau.jpg)

Gap只在**RR**事务隔离级别存在

## 记录锁（Record）

![120910194139083](http://ww4.sinaimg.cn/large/006tNc79gy1g5xaqhyjmzj30pw08sta3.jpg)

# 怎么利用锁解决脏读、不可重复读、幻读

![120910194139087](http://ww1.sinaimg.cn/large/006tNc79gy1g5xaqjrcibj30t00c9ju4.jpg)



![120910194139091](http://ww4.sinaimg.cn/large/006tNc79gy1g5xaqm5m0rj30sa0cg774.jpg)



![120910194139095](http://ww4.sinaimg.cn/large/006tNc79gy1g5xaqoelwzj30sz0bsadq.jpg)

# 死锁介绍

多个并发事务（2个或者以上）；

每个事务都持有锁（或者是已经在等待锁）;

每个事务都需要再继续持有锁；

事务之间产生加锁的循环等待，形成死锁。

**小结**:我在等你、你在等我。



## 死锁如何避免

1. 类似的业务逻辑以固定的顺序访问表和行。
2. 大事务拆小。大事务更倾向于死锁，如果业务允许，将大事务拆小。
3. 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁概率。
4. 降低隔离级别，如果业务允许，将隔离级别调低也是较好的选择
5. 为表添加合理的索引。可以看到如果不走索引将会为表的每一行记录添加上锁（或者说是表锁）

