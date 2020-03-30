# Mycat是什么

- 一个彻底开源的，面向企业应用开发的大数据库集群
- 支持事务、ACID、可以替代MySQL的加强版数据库
- 一个可以视为MySQL集群的企业级数据库，用来替代昂贵的Oracle集群
- 一个融合内存缓存技术、NoSQL技术、HDFS大数据的新型SQL Server
- 结合传统数据库和新型分布式数据仓库的新一代企业级数据库产品
- 一个新颖的数据库中间件产品

![1545233781118](http://ww1.sinaimg.cn/large/006tNc79gy1g5xbvvbmwdj30fs0aedgq.jpg)

通俗点讲，应用层可以将它看作是一个数据库的代理（或者直接看成加强版数据库）

# Mycat名称解释

1. 逻辑库  `db_user` `db_store`

2. 逻辑表

   1. 分片表 `用户表`

      用户表按照UID取模分成两片

   2. 全局表    `数字字典表`

      数据字典，比如存放用户的会员等级。

   3. ER表  `用户地址表`

      每个用户，都有多个地址，ER表将按照用户的分片规则，和用户能匹配上的地址被分到同一个片上

   4. 非分片表  `门店表`，`店员表`

      不需要分片的，数据量较小，和其他表没有关联关系

3. 分片规则    如： userID%2（按照userID取2的模）	

4. 节点  

   节点主机（写、读节点主机）

![121723051595077](http://ww2.sinaimg.cn/large/006tNc79gy1g5xbw74gllj30zv0q4dk2.jpg)

# Mycat目录

## windows

![121723051595088](http://ww1.sinaimg.cn/large/006tNc79gy1g5xbwbvq8qj30h105gdh7.jpg)

## linux

```mysql
[root@localhost mycat]# pwd
/software/mycat
[root@localhost mycat]# ll
total 12
drwxr-xr-x 2 root root  190 Dec 17 10:16 bin
drwxrwxrwx 2 root root    6 Feb 29  2016 catlet
drwxrwxrwx 4 root root 4096 Dec 17 10:16 conf
drwxr-xr-x 2 root root 4096 Dec 17 10:16 lib
drwxrwxrwx 2 root root    6 Oct 28  2016 logs
-rwxrwxrwx 1 root root  217 Oct 28  2016 version.txt
[root@localhost mycat]# 
```

## 目录解释

- **bin** 程序目录，存放了 window 版本和 linux 版本可执行文件./mycat {start|restart|stop|status…}
- **conf** 目录下存放配置文件
  - **server.xml** 是 Mycat 服务器参数调整和用户授权的配置文件
  - **schema.xml** 是逻辑库定义和表
  - **rule.xml** 是分片规则的配置文件，分片规则的具体一些参数信息单独存放为文件，也在 这个目录下
  - **log4j2.xml**配置logs目录日志输出规则  wrapper.conf JVM相关参数调整
- **lib** 目录下主要存放 mycat 依赖的一些 jar 文件
- **logs**目录日志存放日志文件

# Mycat配置文件详解

## schema.xml（逻辑库表）

dataHost

​	balance：

​	负载/读写分离 均衡类型。

​	取值为

​	balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的 writeHost 上。

​	balance="1"，全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且 M1 与 M2 互为主备)，正常情况下，M2,S1,S2 都参与 select 语句的负载均衡。

​	balance="2"，所有读操作都随机的在 writeHost、readhost 上分发。

 

​	switchType 

​	主从切换策略

​	-1 表示不自动切换

​	1 默认值，自动切换

​	2 基于 MySQL 主从同步的状态决定是否切换

​		心跳语句设置为 show slave status

​	通过检测 show slave status 中的 "Seconds_Behind_Master", 			"Slave_IO_Running" "Slave_SQL_Running" 三个字段以及slaveThreshold 设置的值进行判断是否进行主从切换

​	

​	usingDecrypt 

​	可加入usingDecrypt属性来指定密码加密。1开启，0不开启

​	进入lib目录的Mycat-server-1.6-RELEASE.jar 执行

​	java -cp Mycat-server-1.6-RELEASE.jar io.mycat.util.DecryptUtil 		1:host:user:password

​	1:host:user:password 中 1 为 db 端加密标志，host 为 dataHost 的 host 名称

## server.xml（启动参数）

**sequnceHandlerType**  

​	全局序列的方式

[^全局序列]: 文后末尾

firewall

​	黑白名单设置

user

​	benchmark  当前端连接达到设置的值，不再允许这个用户进行接入

​	表级的DML权限控制。4位2进制   顺序为 insert update select delete

## rule.xml（拆分规则）

|      | 连续分片                                              | 离散分片                                                     | 综合类分片                        |
| ---- | ----------------------------------------------------- | ------------------------------------------------------------ | --------------------------------- |
| 优点 | 扩容无需迁移数据，范围条件查询资源消耗小              | 数据分布均匀，并发能力强，不受限分片节点                     | 二者兼并                          |
| 缺点 | 数据热点问题，并发能力受限于分片节点                  | 移植性差，扩容难                                             | 二者兼并                          |
| 代表 | 按日期（天）分片<br>自定义数字范围分片<br/>自然月分片 | 枚举分片<br/> 数字取模分片<br/> 字符串hash分片<br/> 一致性哈希分片<br/>程序指定 | 范围求模分片<br/>取模范围约束分片 |

### 连续分片

#### 自定义数字范围分片

提前规划好分片字段某个范围属于哪个分片

```xml
<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
	<property name="mapFile">autopartition-long.txt</property>
	<property name="defaultNode">0</property>
</function>
```

`defaultNode` 超过范围后的默认节点。

此配置非常简单，即预先制定可能的id范围到某个分片
0-500M=0
500M-1000M=1
1000M-1500M=2
或
0-10000000=0
10000001-20000000=1
注意： 所有的节点配置都是从0开始，及0代表节点1

#### 按日期（天、月）分片

**按日期（天）分片**： 从开始日期算起，按照天数来分片

```xml
<function name="sharding-by-date" class="io.mycat.route.function.PartitionByDate">
	<property name="dateFormat">yyyy-MM-dd</property> <!—日期格式-->
	<property name="sBeginDate">2014-01-01</property> <!—开始日期-->
	<property name="sPartionDay">10</property> <!—每分片天数-->
</function>
```

**按日期（自然月）分片**： 从开始日期算起，按照自然月来分片

```xml
<function name="sharding-by-month" class="io.mycat.route.function.PartitionByMonth">
	<property name="dateFormat">yyyy-MM-dd</property> <!--日期格式-->
	<property name="sBeginDate">2014-01-01</property> <!--开始日期-->
</function>
```

注意： 需要提前将分片规划好，建好，否则有可能日期超出实际配置分片数

#### 按单月、小时分片

按单月小时分片：最小粒度是小时，可以一天最多24个分片，最少1个分片，一个月完后下月从头开始循环。

```xml
<function name="sharding-by-hour" class="io.mycat.route.function.LatestMonthPartion">
	<property name="splitOneDay">24</property> <!-- 将一天的数据拆解成几个分片-->
</function>
```

注意事项：每个月月尾，需要手工清理数据

### 离散分片

#### 枚举分片

**枚举分片**：通过在配置文件中配置可能的枚举id，自己配置分片，本规则适用于特定的场景，比如有些业务需要按照省份或区县来做保存，而全国省份区县固定的

```xml
<function name="hash-int" class="io.mycat.route.function.PartitionByFileMap">
	<property name="mapFile">partition-hash-int.txt</property>
	<property name="type">0</property>
	<property name="defaultNode">0</property>
</function>
```

`partition-hash-int.txt` 配置：
10000=0
10010=1

mapFile标识配置文件名称
type默认值为0（0表示Integer，非零表示String）
默认节点的作用：枚举分片时，如果碰到不识别的枚举值，就让它路由到默认节点

#### 十进制取模

十进制求模分片：规则为对分片字段十进制取模运算。数据分布最均匀

```xml
<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
	<!-- how many data nodes -->
	<property name="count">3</property>
</function>
```

#### 应用指定分片

应用指定分片：规则为对分片字段进行字符串截取，获取的字符串即指定分片。

```xml
<function name="sharding-by-substring"  class="io.mycat.route.function.PartitionDirectBySubString">
	<property name="startIndex">0</property><!-- zero-based -->
	<property name="size">2</property>
	<property name="partitionCount">8</property>
	<property name="defaultPartition">0</property>
</function>
```

startIndex 开始截取的位置
size 截取的长度
partitionCount 分片数量
defaultPartition 默认分片

例如 id=05-100000002
在此配置中代表根据 id 中从 startIndex=0，开始，截取 siz=2 位数字即 05，05 就是获取的分区，如果没传默认分配到 defaultPartition

#### 字符串截取数字hash分片

截取数字hash分片
此规则是截取字符串中的int数值hash分片

```xml
<function name="sharding-by-stringhash" class="io.mycat.route.function.PartitionByString">
	<property name=length>512</property><!-- zero-based -->
	<property name="count">2</property>
	<property name="hashSlice">0:2</property>
</function>
```

length代表字符串hash求模基数，count分区数，其中length*count=1024

hashSlice hash预算位，即根据子字符串中int值 hash运算

0 代表 str.length(), -1 代表 str.length()-1，大于0只代表数字自身

可以理解为substring（start，end），start为0则只表示0

例1：值“45abc”，hash预算位0:2 ，取其中45进行计算

例2：值“aaaabbb2345”，hash预算位-4:0 ，取其中2345进行计算

#### 一致性Hash分片

此规则优点在于扩容时迁移数据量比较少，前提分片节点比较多，虚拟节点分配多些。

虚拟节点少的缺点是会造成数据分布不够均匀

如果实际分片数量比较少，迁移量会比较多

```xml
<function name="murmur" class="io.mycat.route.function.PartitionByMurmurHash">
	<property name="seed">0</property><!-- 创建hash对象的种子，默认0-->
	<property name="count">2</property><!-- 要分片的数据库节点数量，必须指定，否则没法分片-->
	<property name="virtualBucketTimes">160</property>
</ function>
```

注意：
一个实际的数据库节点被映射为这么多虚拟节点，默认是160倍，也就是虚拟节点数是物理节点数的160倍

##### 理解一致性hash

1. hash(str){return 0-> 2^32},将整个0-2^32的hash值，作为一个hash环。
2. 取node唯一标示计算出hash值，该hash的结果即node在整个hash环中的位置。
3. 将数据进行hash计算之后，顺时针找对应的node，该node即为该数据的服务。

如下图所示

![121723051595129](http://ww3.sinaimg.cn/large/006tNc79gy1g5xbwka29yj30hz0ddgq0.jpg)

当node节点增加时，影响的范围将大量减少。如下图所示

![121723051595133](http://ww4.sinaimg.cn/large/006tNc79gy1g5xbwmyhsnj30jk0djwjk.jpg)

### 综合分片

#### 范围求模分片

范围求模分片：先进行范围分片计算出分片组，组内再求模。

优点可以避免扩容时的数据迁移，又可以一定程度上避免范围分片的热点问题

分片组内使用求模可以保证组内数据比较均匀，分片组之间是范围分片可以兼顾范围查询。

最好事先规划好分片的数量，数据扩容时按分片组扩容，则原有分片组的数据不需要迁移。由于分片组内数据比较均匀，所以分片组内可以避免热点数据问题。

```xml

```

partition-range-mod.txt
以下配置一个范围代表一个分片组，=号后面的数字代表该分片组所拥有的分片的数量。
0-200M=5 //代表有5个分片节点
200M-400M=6
400M-600M=6
600M-800M=8
800M-1000M=7

#### 取模范围约束分片

对指定分片列进行取模后再由配置决定数据的节点分布。

```xml
<function name="sharding-by-pattern" class="io.mycat.route.function.PartitionByPattern">
	<property name="patternValue">256</property>
	<property name="defaultNode">2</property>
	<property name="mapFile">partition-pattern.txt</property>
</function>
```

patternValue 即求模基数，
defaoultNode 默认节点
partition-pattern.txt配置
1-32=0
33-64=1
65-96=2
97-128=3
128-256=4
配置文件中，1-32 即代表id%256后分布的范围。
如果id非数字，则会分配在defaoultNode 默认节点

### 分片取舍

数据特点：
活跃的数据热度较高规模可以预期，增长量比较稳定

![1545239169634](http://ww3.sinaimg.cn/large/006tNc79gy1g5xbwr7ttpj30na0a3gmh.jpg)

数据特点：
活跃的数据为历史数据，热度要求不高。规模可以预期，增长量比较稳定. 优势可定时清理或者迁移数据

![1545239488121](http://ww4.sinaimg.cn/large/006tNc79gy1g5xbwuin7jj30dt086js1.jpg)

### 分片选择总结

1. 根据业务数据的特性合理选择分片规则
2. 善用全局表、ER关系表解决join操作
3. 用好primaryKey让你的性能起飞

## wrapper.conf（jvm内核调整）

//TODO





# 全局序列

1.本地文件方式

​	sequnceHandlerType  = 0
​	配置sequence_conf.properties
​	使用next value for MYCATSEQ_XXX

2.数据库方式（见附录）

​	sequnceHandlerType = 1
​	配置sequence_db_conf.properties
​	使用next value for MYCATSEQ_XXX或者指定autoIncrement

3.本地时间戳方式

​	ID= 64 位二进制 (42(毫秒)+5(机器 ID)+5(业务编码)+12(重复累加)
​	sequnceHandlerType = 2
​	配置sequence_time_conf.properties
 	指定autoIncrement

4, 程序方式
​	 Snowflake、UUID、Redis 、...

# Mycat之注解

**概念：** 

​	MyCat对自身不支持的Sql语句提供了一种解决方案——在要执行的SQL语句前添加额外的一段由注解SQL组织的代码，这样Sql就能正确执行，这段代码称之为“注解”。注解的使用相当于对mycat不支持的sql语句做了一层透明代理转发，直接交给目标的数据节点进行sql语句执行，其中注解SQL用于确定最终执行SQL的数据节点。注解的形式是：

```mysql
/*!mycat: sql=注解Sql语句*/
```

注解的使用方式是：

```mysql
/*!mycat: sql=注解Sql语句*/真正执行Sql
```

使用时将=号后的“注解Sql语句”替换为需要的Sql语句即可。 

```mysql
/*!mycat: sql=select * from users where userID=1*/ select fun() from dual;
/*!mycat: sql=select * from users where userID=1*/ CALL proc_test();
/*!mycat: sql=select * from users where userID=1*/ insert into users(id,name) select
id,name from otherUsers;
/*! mycat: sql=select 1 from users*/ create table test2(id int);
/*!mycat: db_type=slave*/ select * from employee
```

# Mycat关联查询总结

1. 用好ER表

2. 善用全局表

3. 使用注解

   ```mysql
   /*!mycat:catlet=io.mycat.catlets.ShareJoin */
   select * from users u,employee em on u.phoneNum=em.phoneNum where u.phoneNum='13633333333' ;
   ```

# Mycat命令行监控工具

- 重载配置文件
- 查看运行状态
- 提供性能数据

```mysql
mysql -uuser –ppwd -P9066
show @@help 查看所有命令
reload @@config
reload @@config_all
show @@database
show @@datanode
show @@datasource
show @@cache
show @@connection
show @@connection.sql
show @@backend
kill @@connection id1，id2
show @@heartbeat
show @@sysparam
```

# Mycat弱XA事务机制


![1545315485461](http://ww3.sinaimg.cn/large/006tNc79gy1g5xbx9ybsbj30lw0fb0tw.jpg) 为什么2PC提交.1是2PC才会有事务管理器统一管理的机会；2尽可能晚地提交事务，让事务在提交前尽可能地完成所有能完成的工作，这样，最后的提交阶段将是耗时极短，耗时极短意味着操作失败的可能性也就降低

XA 是一个两阶段提交协议，规定事务协调/管理器和资源管理器接口

二阶段提交协议为了保证事务的一致性，不管是事务管理器还是各个资源管理器，每执行一步操作，都会记录日志，为出现故障后的恢复准备依据

Mycat 第二阶段的提交没有做相关日志的记录，所以说他是一个弱XA的分布式事务解决方案 |



# Mycat之mysqldump方式进行快速移植

![1545315645493](http://ww2.sinaimg.cn/large/006tNc79gy1g5xbxvhbduj30vt0ev0tk.jpg)

mysqldump方式

导出数据

- mysqldump -uroot -p123456 -h192.168.8.137 -c db_user_old users > users.sql

ER子表

- mysqldump -uroot -p123456 -h192.168.8.137 -c --skip-extended-insert db_user_old user_address > userAddress.sql

导入

- mysql -h192.168.8.151 -uroot -p123456 -P8066  -f db_user < users.sql

- mysql -h192.168.8.151 -uroot -p123456 -P8066  -f db_user < userAddress.sql

# 附录

全局序列-数据库方式需要三个函数，步骤如下

```mysql
-- 1.选定一个节点，创建 MYCAT_SEQUENCE 表
DROP TABLE IF EXISTS mycat_sequence;
CREATE TABLE MYCAT_SEQUENCE (name VARCHAR(50) NOT NULL,current_value INT NOT
NULL,increment INT NOT NULL DEFAULT 100, PRIMARY KEY(name)) ENGINE=InnoDB;

-- 2,插入具有自增ID的表名记录
INSERT INTO MYCAT_SEQUENCE(name,current_value,increment) VALUES ('USERS', 100000, 100);


-- 3,创建3个函数

-- 传入序列名得到当前的序列的值
DROP FUNCTION IF EXISTS mycat_seq_currval;

CREATE FUNCTION mycat_seq_currval(seq_name VARCHAR(50)) RETURNS varchar(64)
BEGIN
	DECLARE retval VARCHAR(64);
	SET retval='-999999999,null';
	SELECT concat(CAST(current_value AS CHAR),',',CAST(increment AS CHAR)) INTO retval FROM
	MYCAT_SEQUENCE WHERE name = seq_name;
	RETURN retval;
END



-- 给指定的序列设定值
DROP FUNCTION IF EXISTS mycat_seq_setval;

CREATE FUNCTION mycat_seq_setval(seq_name VARCHAR(50),value INTEGER) RETURNS varchar(64)
BEGIN
	UPDATE MYCAT_SEQUENCE
	SET current_value = value
	WHERE name = seq_name;
	RETURN mycat_seq_currval(seq_name);
END


-- 获取传入序列名的 一段序列  如 100,100
DROP FUNCTION IF EXISTS mycat_seq_nextval;
CREATE FUNCTION mycat_seq_nextval(seq_name VARCHAR(50)) RETURNS varchar(64)
BEGIN
	UPDATE MYCAT_SEQUENCE SET current_value = current_value + increment WHERE name = seq_name;
	RETURN mycat_seq_currval(seq_name);
END


--4，在指定的schema中加入mycat_sequence表
<table name = "mycat_sequence" dataNode ="db_user_dataNode2" />
```
