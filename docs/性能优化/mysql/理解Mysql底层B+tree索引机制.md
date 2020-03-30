# 理解Mysql底层B+tree索引机制
## 初识Mysql体系结构

**整体结构图**

![img](http://ww4.sinaimg.cn/large/006tNc79gy1g5xaumvwtgj30ne0d642s.jpg)

- Connectors 

  接入方支持协议很多

- Management Serveices & Utilities

  系统管理和控制工具，例如：备份恢复，mysql复制集群等

- Connection Pool

  连接池：管理缓冲用户连接、用户名、密码、权限校验、线程处理等需要缓存的需求

- SQL Interface

  SQL接口：接受用户的SQL命令，并且返回用户需要查询的结果。比如select from就是调用SQL Interface

- Parser

   解析器，SQL命令传递到解析器的时候会被解析器验证和解析。解析器是由Lex和YACC实现的。

- Optimizer

  查询优化器，SQL语句在查询之前会使用查询优化器对查询进行优化

- Cache和Buffer（高速缓存区）

   查询缓存，如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。

- Pluggable Storage Engines

  插件式存储引擎。存储引擎是MySql中具体的与文件打交道的子系统。也是Mysql最具有特色的一个地方。 Mysql的存储引擎是插件式的。

- File System

  文件系统，数据、日志（redo，undo）、索引、错误日志、查询记录、慢查询等

**运行时机理图**

![img](http://ww4.sinaimg.cn/large/006tNc79gy1g5xaupfwmdj30o60ddkjl.jpg)

## 理解Mysql底层B+tree索引机制

写在前面的话

**正确**的创建**合适**的索引，是提升数据库查询性能的**基础**

### **索引是什么？**

索引定义：

索引是为了加速对表中数据行的检索而创建的一种分散存储的**数据结构**

![120421194082031](http://ww2.sinaimg.cn/large/006tNc79gy1g5xauszguoj30uq0910vl.jpg)

### 为什么要用索引？

索引意义：

​	索引能极大的减少存储引擎需要扫描的数据量

​	索引可以把随机IO变成顺序IO

​	索引可以帮助我们在进行分组、排序等操作时，避免使用临时表

### 为什么是B+Tree？

二叉查找树、平衡二叉树、多路平衡查找树、加强版多路平衡查找树 

详细了解上面的数据结构可以参见[帖子](https://www.cnblogs.com/vianzhang/p/7922426.html)

#### 二叉查找树，Binary Search Tree

每个分支最多有两个（路）

![120421194082038](http://ww3.sinaimg.cn/large/006tNc79gy1g5xauy5su0j30i30h10ur.jpg)

#### 平衡二叉查找树，Balance binary search tree

平衡二叉树（AVL树）在符合二叉查找树的条件下，还满足任何节点的两个子树的高度最大差为1。

先了解下磁盘的相关知识。

系统从磁盘读取数据到内存时是以磁盘块（block）为基本单位的，位于同一个磁盘块中的数据会被一次性读取出来，而不是需要什么取什么。



```
X < 10 --> P1
X = 10 --> 命中
X > 10 --> P2
```



![120421194082041](http://ww1.sinaimg.cn/large/006tNc79gy1g5xav6q2pqj30so0gzwh2.jpg)

一个磁盘块：

> ​	绿色：关键字 （这里关键字就比如是id = 10）
>
> ​	蓝色：数据区（比如图上为 id  为 10 整条记录 id 、name 、age  、 phoneNum）
>
> ​	粉色：P1 P2 子节点的引用
>

模拟查找关键字id = 8的过程：

> ​	1.从根节点开始找磁盘块1，读入内存，  发现  8 < 10 --> P1 ，一次IO操作
>
> ​	2.此时P1指向磁盘块2，读入内存，  	     发现 8 > 5 --> P2， 一次IO操作
>
> ​	3.此时P2指向磁盘块5，读入内存，	     发现 8 = 8，命中 ，一次IO操作
>

（这里的IO操作理解为读取一个磁盘块，包括关键字、数据区、子节点引用）

上面索引到id为8的记录，需要3次IO操作、3次内存查找。毕竟上面是举例子，假如我们的老师teacher表数据量非常大，那id就很多，这样这个平衡二叉树的高度就很大，假如很不幸我们要找的id = 8的记录，在这颗树最大的高度，那这样的命中索引，需要很多次IO操作。

可想而知，这种索引的规则太随机，MySQL肯定不是采用的这样

**小结**

它太深了
​	数据处的（高）深度决定着他的IO操作次数，IO操作耗时大
它太小了
​	每一个磁盘块（节点/页）保存的数据量太小了
​	没有很好的利用操作磁盘IO的数据交换特性，
​	也没有利用好磁盘IO的预读能力（空间局部性原理），从而带来频繁的IO操作

### 多路平衡查找树，B-Tree

B-Tree是为磁盘等外存储设备设计的一种平衡查找树。



***绝对平衡树*** 

B-Tree中的每个节点根据实际情况可以包含大量的关键字信息和分支，如下图所示为一个3阶的B-Tree： 

```
X < 17 --> P1   		
X = 17 命中
17< X < 35 --> P2 	
X = 35 命中
X > 35 --> P3
```



![120421194082048](http://ww2.sinaimg.cn/large/006tNc79gy1g5xav9e3yej30yo0hjdkz.jpg)

模拟查找关键字的过程：(省略)

B-Tree相对于AVLTree缩减了节点个数，使每次磁盘I/O取到内存的数据都发挥了作用，从而提高了查询效率。

### 加强版多路平衡查找树 B+Tree

B+Tree是在B-Tree基础上的一种优化，使其更适合实现外存储索引结构，InnoDB存储引擎就是用B+Tree实现其索引结构。

MySQL的B+Tree

左闭合B+Tree：

```
1	<= X < 28	--> P1
28	<= X < 66	--> P2
66	<= X		--> P3
```

![120421194082052](http://ww1.sinaimg.cn/large/006tNc79gy1g5xavdx89fj30uo0grtex.jpg)

从上一节中的B-Tree结构图中可以看到每个节点中不仅包含数据的key值（关键字，子节点引用），还有data值（一整条记录的磁盘地址）。而每一个页的存储空间是有限的，如果data数据较大时将会导致每个节点（即一个页）能存储key的数量很小，当存储的数据量很大时同样会导致B-Tree的深度较大，增大查询时的磁盘I/O次数，进而影响查询效率。

**在B+Tree中，所有数据记录节点都是按照键值大小顺序存放在同一层的叶子节点上，而非叶子节点上只存储key值信息，这样可以大大加大每个节点存储的key值数量，降低B+Tree的高度。**

**B+Tree与B-Tree的区别**

1. B+节点关键字搜索采用闭合区间
2. B+非叶节点不保存数据相关信息，只保存关键字和子节点的引用
3. B+关键字对应的数据保存在叶子节点中
4. B+叶子节点是顺序排列的，并且相邻节点具有顺序引用的关系



### 为什么选用B+Tree ？

B+树是B-树的变种（PLUS版）多路绝对平衡查找树，他拥有B-树的优势

B+树扫库、扫表能力更强

B+树的磁盘读写能力更强

B+树的排序能力更强

B+树的查询效率更加稳定（仁者见仁、智者见智）

## Mysql B+Tree索引体现形式

### Myisam

MyISAM是默认[存储引擎](https://baike.baidu.com/item/%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E)（Mysql5.5前）。它基于更老的ISAM代码，但有很多有用的扩展。 

每个MyISAM在磁盘上存储成三个文件，每一个文件的名字均以表的名字开始，扩展名指出文件类型。

`.frm`文件存储表定义；

`·MYD` (`MYData`)文件存储表的数据；

`.MYI` (`MYIndex`)文件存储表的索引。

![120421194082063](http://ww4.sinaimg.cn/large/006tNc79gy1g5xavhraa6j310t0jpwkk.jpg)

MyISAM的索引与行记录是分开存储的，叫做**非聚集索引**（UnClustered Index）。



其主键索引与普通索引没有本质差异：

> - 有连续聚集的区域单独存储行记录
> - 主键索引的叶子节点，存储主键，与对应行记录的指针
> - 普通索引的叶子结点，存储索引列，与对应行记录的指针
>

*画外音：MyISAM的表可以没有主键。*

 

主键索引与普通索引是两棵独立的索引B+树，通过索引列查找时，先定位到B+树的叶子节点，再通过指针定位到行记录。

![120421194082067](../../images/optimize/mysql/120421194082067.Png)

### Innodb

InnoDB，是MySQL的数据库引擎之一，为[MySQL AB](https://baike.baidu.com/item/MySQL%20AB)发布binary的标准之一。InnoDB由Innobase Oy公司所开发，2006年五月时由[甲骨文公司](https://baike.baidu.com/item/%E7%94%B2%E9%AA%A8%E6%96%87%E5%85%AC%E5%8F%B8/430115)并购。与传统的[ISAM](https://baike.baidu.com/item/ISAM)与[MyISAM](https://baike.baidu.com/item/MyISAM)相比，InnoDB的最大特色就是支持了[ACID](https://baike.baidu.com/item/ACID/10738)兼容的[事务](https://baike.baidu.com/item/%E4%BA%8B%E5%8A%A1/5945882)（Transaction）功能，类似于[PostgreSQL](https://baike.baidu.com/item/PostgreSQL)。

每个InnoDB在磁盘上存储成2个文件，每一个文件的名字均以表的名字开始，扩展名指出文件类型。

`.frm` 文件存储表定义；

`·ibd` 数据与索引文件；

![120421194082071](http://ww4.sinaimg.cn/large/006tNc79gy1g5xavlthr9j30pd0jm410.jpg)

简介（[百度百科](https://baike.baidu.com/item/innodb/8970025?fr=aladdin)）

​	事务型数据库的首选引擎，支持ACID事务，支持行级锁定。InnoDB是为处理巨大数据量时的最大性能设计。InnoDB存储引擎完全与MySQL服务器整合，InnoDB存储引擎为在主内存中缓存数据和索引而维持它自己的缓冲池。InnoDB存储它的表&索引在一个表空间中，表空间可以包含数个文件（或原始磁盘分区）。这与MyISAM表不同，比如在MyISAM表中每个表被存在分离的文件中。InnoDB 表可以是任何尺寸，即使在文件尺寸被限制为2GB的操作系统上。InnoDB默认地被包含在MySQL二进制分发中。Windows Essentials installer使InnoDB成为Windows上MySQL的默认表。

![120421194082075](http://ww3.sinaimg.cn/large/006tNc79gy1g5xavoutahj31ah0kxgq1.jpg)

MySQL的Innodb是以主键索引来组织数据结构的，**主键索引**与行记录是存储在一起的，故叫做**聚集索引**（Clustered Index）

因为这个特性，InnoDB的表必须要有聚集索引：

> 1、如果表定义了PK，则PK就是聚集索引；
>
> 2、如果表没有定义PK，则第一个非空unique列是聚集索引；
>
> 3、否则，InnoDB会创建一个隐藏的row-id作为聚集索引；

InnoDB的**聚集索引**，也只能够有一个，因为数据行在物理磁盘上只能有一份聚集存储。

InnoDB的**普通索引**，可以有多个，它与聚集索引是不同的。



InnoDB存储引擎中页的大小默认为16KB，一般表的主键类型为INT（占用4个字节）或BIGINT（占用8个字节），指针类型也一般为4或8个字节，也就是说一个页（B+Tree中的一个节点）中大概存储16KB/(8B+8B)=1K个键值（因为是估值，为方便计算，这里的K取值为〖10〗^3）。也就是说一个深度为3的B+Tree索引可以维护10^3 * 10^3 * 10^3 = 10亿 条记录。

实际情况中每个节点可能不能填充满，因此在数据库中，B+Tree的高度一般都在2~4层。[mysql](http://lib.csdn.net/base/mysql)的InnoDB存储引擎在设计时是将**根节点常驻内存**的，也就是说查找某一键值的行记录时最多只需要1~3次磁盘I/O操作。

数据库中的B+Tree索引可以分为**聚集索引（clustered index）**和**辅助索引（secondary index）**。上面的B+Tree示例图在数据库中的实现即为聚集索引，**聚集索引的B+Tree中的叶子节点存放的是整张表的行记录数据**。辅助索引与聚集索引的区别在于**辅助索引的叶子节点并不包含行记录的全部数据，而是存储相应行数据的聚集索引键，即主键**。当通过辅助索引来查询数据时，InnoDB存储引擎会遍历辅助索引找到主键，然后再通过主键在聚集索引中找到完整的行记录数据。



### Innodb VS Myisam

![120421194082079](http://ww3.sinaimg.cn/large/006tNc79gy1g5xavror10j30po0bkwkp.jpg)



Innodb: 假如你频繁的修改、更新辅助索引，主键索引是不需要重排序的

Myisam：主键索引和辅助键索引无区别



## 索引知识补充

列的离散性 count(distinct col) : count(col)

找出离散性最好的列？

![120421194082084](http://ww2.sinaimg.cn/large/006tNc79gy1g5xavvsws9j30jl0gkdhm.jpg)

name的离散性最大，越大离散性越好
结论：
​	离散性越高
​	选择性就越好

### 最左匹配原则

对索引中关键字进行计算（对比），一定是从左往右依次进行，且不可跳过

![120421194082088](http://ww3.sinaimg.cn/large/006tNc79gy1g5xavz755ij30l70endh5.jpg)

### 注意避免冗余索引

冗余索引指的是索引的功能相同，能够命中 就肯定能命中 ，那么 就是冗余索引如（name,city ）和（nam个索引就是冗余索引，能够命中后者的查询肯定是能够命中前者的 在大多数情况下，都应该尽量扩展已有不是创建新索引。

MySQLS.7 版本后，可以通过查询 sys 库的 `schemal_r dundant_indexes` 表来查看冗余索引Mysql如何为表字段添加索引？？？

1.添加PRIMARY KEY（主键索引）

```mysql
ALTER TABLE `table_name` ADD PRIMARY KEY ( `column` )
```

2.添加UNIQUE(唯一索引)

```mysql
ALTER TABLE `table_name` ADD UNIQUE ( `column` )
```

3.添加INDEX(普通索引)

```mysql
ALTER TABLE `table_name` ADD INDEX index_name ( `column` )
```

4.添加FULLTEXT(全文索引)

```mysql
ALTER TABLE `table_name` ADD FULLTEXT ( `column`)
```

5.添加多列索引

```mssql
ALTER TABLE `table_name` ADD INDEX index_name ( `column1`, `column2`, `column3`)
```

### 联合索引？so seay

单列索引				节点中关键字[name]

联合索引				节点中关键字[name,phoneNum]
**单列索引是特殊的联合索引**

联合索引列选择原则

1. 经常用的列优先 【最左匹配原则】
2. 选择性（离散度）高的列优先【离散度高原则】
3. 宽度小的列优先【最少空间原则】

**机灵的李二狗**

经排查发现最常用的sql语句：

```mysql
Select * from tbl_user where name = ? ;
Select * from tbl_user where name = ? and phoneNum = ?;
```

机灵的李二狗的解决方案：

```mysql
create index idx_name on tbl_user(name);#冗余索引，根本用不上
create index idx_name_phoneNum on tbl_user(name,phoneNum);
```

### 覆盖索引

如果查询列可通过索引节点中的关键字直接返回，则该索引称之为**覆盖索引**。
覆盖索引可减少数据库IO，将随机IO变为顺序IO，可提高查询性能

例：

```mysql
create index idx_name_phoneNum on tbl_user(name,phoneNum);

select name,phoneNum from tbl_user where name = "张三"；
```

在索引的`子节点`命中了 返回的列，就不需要再继续往B+Tree的底部叶子节点 读取数据了，减少了IO操作，提高了查询速度。

这就是为什么不建议大家select * from table 的原因，需要什么列就查什么列，可能会命中覆盖索引，这样就会极大的提高查询效率。

**这里补充一个误区**

![1549164072336](http://ww1.sinaimg.cn/large/006tNc79gy1g5xaxqx8yyj30u00i9ac1.jpg)

`如果查询条件是 LIKE 'abc%‘，MySQL 将使用索引；如果查询条件是 LIKE '%abc’，MySQL 将不使用索引。`

`错误、错误、错误`，只能说这句话不严谨

比如以上面的例子

```mysql
select name,phoneNum from tbl_user where name like "%张三%";#这个是会走索引的
```

如下图所示，执行一下EXPLAIN

![1549164782958](http://ww2.sinaimg.cn/large/006tNc79gy1g5xay4bq26j30ns0bawez.jpg)

## 现在，你能都理解了么？

索引列的数据长度能少则少。



索引一定不是越多越好，越全越好，一定是建合适的。



匹配列前缀可用到索引 like 9999%（如果索引列数据离散性太差，执行器可能会放弃索引，全盘扫描），like %9999%、like %9999大部分情况用不到索引（特殊情况命中覆盖索引）；



Where 条件中 not in 和 <>非主键列操作无法使用索引；主键列操作可以使用索引



匹配范围值，order by 也可用到索引；



多用指定列查询，只返回自己想到的数据列，少用select *；



联合索引中如果不是按照索引最左列开始查找，无法使用索引；



联合索引中精确匹配最左前列并范围匹配另外一列可以用到索引；



联合索引中如果查询中有某个列的范围查询，则其右边的所有列都无法使用索引；