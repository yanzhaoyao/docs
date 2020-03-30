# mysql 查询优化-查询执行的路径

![120622313511055](http://ww2.sinaimg.cn/large/006tNc79gy1g5xau1cb1ij30rn0l644l.jpg)

1. mysql 客户端/服务端通信
2. 查询缓存
3. 查询优化处理
4. 查询执行引擎
5. 返回客户端

# mysql 客户端/服务端通信

Mysql客户端与服务端的通信方式是“半双工”；

**全双工**：双向通信，发送同时也可以接收

**半双工**：双向通信，同时只能接收或者是发送，无法同时做操作

**单工**：只能单一方向传送

**半双工通信**：
在任何一个时刻，要么是有服务器向客户端发送数据，要么是客户端向服务端发送数据，这两个动作不能同时发生。所以我们无法也无需将一个消息切成小块进行传输

**特点和限制**：
客户端一旦开始发送消息，另一端要接收完整个消息才能响应。
客户端一旦开始接收数据没法停下来发送指令。



**Mysql客户端与服务端通信-查询状态**

对于一个mysql连接，或者说一个线程，时刻都有一个状态来标识这个连接正在做什么
查看命令 

`show full processlist / show processlist`

| 说明                 | 截图                                                         |
| -------------------- | ------------------------------------------------------------ |
| 当前控制台client链接 | ![1544227223850](../../images/optimize/mysql/1544227223850.png) |
| 本地Navicat链接      | ![1544227253819](../../images/optimize/mysql/1544227253819.png) |

 https://dev.mysql.com/doc/refman/5.7/en/general-thread-states.html (状态全集)

| Command        | Description                |
| -------------- | -------------------------- |
| Sleep          | 线程正在等待客户端发送数据 |
| Query          | 连接线程正在执行查询       |
| Locked         | 线程正在等待表锁的释放     |
| Sorting result | 线程正在对结果进行排序     |
| Sending data   | 向请求端返回数据           |

可通过kill {id}的方式进行连接的杀掉

![1544227841219](http://ww1.sinaimg.cn/large/006tNc79gy1g5xau8dx8fj30fq06yglq.jpg)

# 查询缓存

**工作原理**：

缓存SELECT操作的结果集和SQL语句；

新的SELECT语句，先去查询缓存，判断是否存在可用的记录集。

**判断标准**：

与缓存的SQL语句，是否完全一样，区分大小写 (简单认为存储了一个key-value结构，key为sql，value为sql查询结果集)

**查询缓存相关的系统变量**：

通过`show variables like '%query_cache%';`

```mysql
mysql> show variables like '%query_cache%';
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| have_query_cache             | YES     |
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 0       |
| query_cache_type             | OFF     |
| query_cache_wlock_invalidate | OFF     |
+------------------------------+---------+
6 rows in set (0.05 sec)

mysql> 
```

`have_query_cache`　　表示这个mysql版本是否支持Query Cache。

`query_cache_limit`　　 允许Cache的单条Query结果集的最大容量，默认是1MB，超过此参数设置的Query结果集将不会被Cache。

`query_cache_min_res_unit`　　设置Query Cache中每次分配内存的最小空间大小，也就是每个Query的Cache最小占用的内存空间大小。

`query_cache_size`　　设置Query Cache所使用的内存大小，默认值为0，大小必须是1024的整数倍，如果不是整数倍，MySQL会自动调整降低最小量以达到1024的倍数。

`query_cache_type`　　控制Query Cache功能的开关，可以设置为0(OFF)，1(ON)和2(DEMAND)三种：0表示关闭Query Cache功能，任何情况下都不会使用Query Cache；1表示开启Query Cache功能，但是当SELECT语句中使用的SQL_NO_CACHE提示后，将不使用Query Cache；2(DEMAND)表示开启Query Cache功能，但是只有当SELECT语句中使用了SQL_CACHE提示后，才使用Query Cache。

`query_cache_wlock_invalidate`　　控制当有写锁加在表上的时候，是否先让该表相关的Query Cache失效，1(TRUE)，在写锁定的同时将使该表相关的所有Query Cache 失效。0(FALSE)，在锁定时刻仍然允许读取该表相关的Query Cache。

`my.cnf`文件中配置参数信息

```
query_cache_type=1
query_cache_size=128M
query_cache_limit=1M
```



------

**show status like 'Qcache%' 命令可查看缓存情况**

```mysql
mysql> show status like 'Qcache%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Qcache_free_blocks      | 0     |
| Qcache_free_memory      | 0     |
| Qcache_hits             | 0     |
| Qcache_inserts          | 0     |
| Qcache_lowmem_prunes    | 0     |
| Qcache_not_cached       | 0     |
| Qcache_queries_in_cache | 0     |
| Qcache_total_blocks     | 0     |
+-------------------------+-------+
8 rows in set (0.01 sec)

mysql> 
```

注：

`Qcache_free_blocks`　　表示查询缓存中目前还有多少剩余的blocks，如果该值显示较大，则说明查询缓存中的内存碎片过多了，可能在一定的时间进行整理。

`Qcache_free_memory`　　查询缓存目前剩余空间大小。

`Qcache_hits`　　　　　   查询缓存的命中次数。

`Qcache_inserts`　　　　查询缓存插入的次数。

`Qcache_lowmem_prunes`   该参数记录有多少条查询因为内存不足而被移除出查询缓存。通过这个值，用户可以适当的调整缓存大小。

`Qcache_not_cached`		表示因为query_cache_type的设置而没有被缓存的查询数量。

`Qcache_queries_in_cache`	当前缓存中缓存的查询数量。

`Qcache_total_blocks`		当前缓存的block数量。

也就是说缓存的命中率为 `Qcache_hits`/(`Qcache_hits``+Qcache_inserts`)

**查询缓存-不会缓存的情况**

1. 当查询语句中有一些不确定的数据时，则不会被缓存。如包含函数NOW()，CURRENT_DATE()等类似的函数，或者用户自定义的函数，存储函数，用户变量等都不会被缓存
2. 当查询的结果大于query_cache_limit设置的值时，结果不会被缓存
3. 对于InnoDB引擎来说，当一个语句在事务中修改了某个表，那么在这个事务提交之前，所有与这个表相关的查询都无法被缓存。因此长时间执行事务，会大大降低缓存命中率
4. 查询的表是系统表
5. 查询语句不涉及到表

**查询缓存-是一个坑？**

为什么mysql默认关闭了缓存开启？？

1. 在查询之前必须先检查是否命中缓存,浪费计算资源
2. 如果这个查询可以被缓存，那么执行完成后，MySQL发现查询缓存中没有这个查询，则会将结果存入查询缓存，这会带来额外的系统消耗
3. 针对表进行写入或更新数据时，将对应表的所有缓存都设置失效。
4. 如果查询缓存很大或者碎片很多时，这个操作可能带来很大的系统消耗

通常项目会在业务层采用Redis、Membercache来缓存数据。

至于说用不用mysql的查询缓存，还是要看业务模型的。

**查询缓存-适用的业务场景**

**以读为主**的业务，数据生成之后就**不常改变**的业务
比如门户类、新闻类、报表类、论坛类、档案类等

# 查询优化处理

查询优化处理的三个阶段：

- 解析sql

  通过lex词法分析,yacc语法分析将sql语句解析成解析树

  https://www.ibm.com/developerworks/cn/linux/sdk/lex/

- 预处理阶段

  根据mysql的语法的规则进一步检查解析树的合法性，如：检查数据的表和列是否存在，解析名字和别名的设置。还会进行权限的验证

- 查询优化器

  优化器的主要作用就是找到最优的执行计划

## **查询优化器如何找到最优执行计划**

- 使用等价变化规则

  5 = 5 and a > 5 改写成 a > 5

  a < b and a = 5 改写成 b > 5 and a = 5

  基于联合索引，调整条件位置等

- 优化count 、min、max等函数

  min函数只需找索引最左边

  max函数只需找索引最右边

  myisam引擎count(*)

- 覆盖索引扫描

- 子查询优化

- 提前终止查询

  用了limit关键字或者使用不存在的条件

- IN的优化

- 先进性排序，再采用二分查找的方式

- ... 

Mysql的查询优化器是基于成本计算的原则。他会尝试各种执行计划。数据抽样的方式进行试验（随机的读取一个4K的数据块进行分析）

## mysql 查询优化-执行计划

表结构见文后附录

```mysql
mysql> EXPLAIN select * from users where id  in  (select userID  from user_address WHERE addr= "上海") \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: <subquery2>
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: <subquery2>.userID
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 3. row ***************************
           id: 2
  select_type: MATERIALIZED
        table: user_address
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5
     filtered: 20.00
        Extra: Using where
3 rows in set, 1 warning (0.31 sec)

mysql> 
```

**执行计划-id**

select查询的序列号，标识执行的顺序

- id相同，执行顺序由上至下
- id不同，如果是子查询，id的序号会递增，id值越大优先级越高（但是也有特殊情况），越先被执行。
- id相同又不同即两种情况同时存在，id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行

**执行计划-select-type**

查询的类型，主要是用于区分普通查询、联合查询、子查询等

- `SIMPLE`：简单的select查询，查询中不包含子查询或者union
- `PRIMARY`：查询中包含子部分，最外层查询则被标记为primary
- `SUBQUERY`/`MATERIALIZED`：SUBQUERY表示在select 或 where列表中包含了子查询
  MATERIALIZED表示where 后面in条件的子查询
- `UNION`：若第二个select出现在union之后，则被标记为union；
- `UNION RESULT`：从union表获取结果的select

**执行计划-table**

查询涉及到的表
直接显示表名或者表的别名

- `<unionM,N>` 由ID为M,N 查询union产生的结果
- `<subqueryN>` 由ID为N查询生产的结果

**执行计划-type`(重点)`**

访问类型，sql查询优化中一个很重要的指标，结果值从好到坏依次是：

**`system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`**

- `system`：表只有一行记录（等于系统表），const类型的特例，基本不会出现，可以忽略不计
- `const`：表示通过索引一次就找到了，const用于比较`primary key` 或者 `unique`索引
- `eq_ref`：唯一索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键 或 唯一索引扫描
- `ref`：非唯一性索引扫描，返回匹配某个单独值的所有行，本质是也是一种索引访问
- `range`：只检索给定范围的行，使用一个索引来选择行
- `index`：Full Index Scan，索引全表扫描，把索引从头到尾扫一遍
- `ALL`：Full Table Scan，遍历全表以找到匹配的行

**执行计划-possible_keys、key、rows、filtered**

**possible_keys**

查询过程中有可能用到的索引

**key**

实际使用的索引，如果为NULL，则没有使用索引

**rows**

根据表统计信息或者索引选用情况，大致估算出找到所需的记录所需要读取的行数

**filtered**

它指返回结果的行占需要读到的行(rows列的值)的百分比

表示返回结果的行数占需读取行数的百分比，filtered的值越大越好

**执行计划-Extra**

十分重要的额外信息

- Using filesort ：

  mysql对数据使用一个外部的文件内容进行了排序，而不是按照表内的索引进行排序读取

- Using temporary：

  使用临时表保存中间结果，也就是说mysql在对查询结果排序时使用了临时表，常见于order by 或 group by

- Using index：

  表示相应的select操作中使用了覆盖索引（Covering Index），避免了访问表的数据行，效率高

- Using where ：

  使用了where过滤条件

- select tables optimized away：

  基于索引优化MIN/MAX操作或者MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段在进行计算，查询执行计划生成的阶段即可完成优化

# 查询执行引擎

调用插件式的存储引擎的原子API的功能进行执行计划的执行

# 返回客户端

- 有需要做缓存的，执行缓存操作

- 增量的返回结果：

  开始生成第一条结果时,mysql就开始往请求方逐步返回数据

  好处： mysql服务器无须保存过多的数据，浪费内存

  用户体验好，马上就拿到了数据

# 测试-执行计划

表结构可以见文末尾附录

## 执行计划-type

### const：

表示通过索引一次就找到了，const用于比较primary key 或者 unique索引。因为只需匹配一行数据，所有很快。如果将主键置于where列表中，mysql就能将该查询转换为一个const?

`explain select * from users where id=1 \G`

```mysql
mysql> explain select * from users where id=1 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.03 sec)

mysql> 
```

### eq_ref：

唯一索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键 或 唯一索引扫描。

`explain select * from users where id in (select userID from user_address WHERE addr= "上海") \G`

```mysql
mysql> explain select * from users where id in (select userID from user_address WHERE addr= "上海") \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: <subquery2>
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: <subquery2>.userID
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 3. row ***************************
           id: 2
  select_type: MATERIALIZED
        table: user_address
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5
     filtered: 20.00
        Extra: Using where
3 rows in set, 1 warning (0.00 sec)

mysql> 
```

### ref：

非唯一性索引扫描，返回匹配某个单独值的所有行。本质是也是一种索引访问，它返回所有匹配某个单独值的行，然而他可能会找到多个符合条件的行，所以它应该属于查找和扫描的混合体

`explain select * from users where uname = '李二狗' \G`

```mysql
mysql> explain select * from users where uname = '李二狗' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: ref
possible_keys: idx_name,idx_union_uname&userLevel&age
          key: idx_name
      key_len: 130
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.09 sec)

mysql> 
```

### range：

只检索给定范围的行，使用一个索引来选择行。key列显示使用了那个索引。一般就是在where语句中出现了`bettween`、`<`、`>`、`in`等的查询。这种索引列上的范围扫描比全索引扫描要好。只需要开始于某个点，结束于另一个点，不用扫描全部索引

`explain select * from users where uname like '李二%' \G`

```mysql
mysql> explain select * from users where uname like '李二%' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: range
possible_keys: idx_name,idx_union_uname&userLevel&age
          key: idx_name
      key_len: 130
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)

mysql> 
```

### index：

Full Index Scan，index与ALL区别为index类型只遍历索引树。这通常为ALL块，应为索引文件通常比数据文件小。（Index与ALL虽然都是读全表，但index是从索引中读取，而ALL是从硬盘读取）

`explain select uname from users \G`

```mysql
mysql> explain select uname from users \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_name
      key_len: 130
          ref: NULL
         rows: 99766
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.02 sec)

mysql> 
```

## 执行计划-Extra

### Using filesort

mysql对数据使用一个外部的文件内容进行了排序，而不是按照表内的索引进行排序读取。也就是说mysql无法利用索引完成的排序操作，成为“文件排序”

`explain select * from users order by lastUpdate desc \G` 

```mysql
mysql> explain select * from users order by lastUpdate desc \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 99766
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)

mysql> 
```

### Using temporary

使用临时表保存中间结果，也就是说mysql在对查询结果排序时使用了临时表，常见于`order by` 和 `group by`

`explain select max(lastUpdate) from users group by lastUpdate \G`

```mysql
mysql> explain select max(lastUpdate) from users group by lastUpdate \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 99766
     filtered: 100.00
        Extra: Using temporary; Using filesort
1 row in set, 1 warning (0.02 sec)

mysql> 
```

### Using index： 

表示相应的select操作中使用了覆盖索引（Covering Index），避免了访问表的数据行，效率高 
`explain select uname from users \G`

```mysql
mysql> explain select uname from users \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_name
      key_len: 130
          ref: NULL
         rows: 99766
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.01 sec)

mysql> 
```

### Using where ：

使用了where过滤

### select tables optimized away：

基于索引优化`MIN`/`MAX`操作或者MyISAM存储引擎优化`COUNT（*）`操作，不必等到执行阶段在进行计算，查询执行计划生成的阶段即可完成优化

`explain select max(id) from users \G` 

```mysql
mysql> explain select max(id) from users \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
   partitions: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: Select tables optimized away
1 row in set, 1 warning (0.00 sec)

mysql> 
```

### no matching row in const table:

执行计划优化阶段，认为const表中没有匹配的行，直接返回结果。

`explain select * from users where id=-1 \G`

```mysql
mysql> explain select * from users where id=-1 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
   partitions: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: no matching row in const table
1 row in set, 1 warning (0.00 sec)

mysql> 
```

# 附录

表结构

```mysql
mysql> desc users;
+------------+-------------+------+-----+---------+----------------+
| Field      | Type        | Null | Key | Default | Extra          |
+------------+-------------+------+-----+---------+----------------+
| id         | int(11)     | NO   | PRI | NULL    | auto_increment |
| uname      | varchar(32) | NO   | MUL | NULL    |                |
| userLevel  | int(11)     | NO   |     | NULL    |                |
| age        | int(11)     | NO   |     | NULL    |                |
| phoneNum   | char(11)    | NO   |     | NULL    |                |
| createTime | datetime    | NO   |     | NULL    |                |
| lastUpdate | datetime    | NO   |     | NULL    |                |
+------------+-------------+------+-----+---------+----------------+
7 rows in set (0.30 sec)

mysql> desc user_address;
+--------+--------------+------+-----+---------+----------------+
| Field  | Type         | Null | Key | Default | Extra          |
+--------+--------------+------+-----+---------+----------------+
| id     | int(11)      | NO   | PRI | NULL    | auto_increment |
| userID | int(11)      | YES  |     | NULL    |                |
| addr   | varchar(256) | YES  |     | NULL    |                |
+--------+--------------+------+-----+---------+----------------+
3 rows in set (0.00 sec)
```

