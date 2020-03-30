# MySQL如何定位慢sql

# MySQL“慢SQL”定位

> 数据库调优我个人觉得必须要明白两件事
>     1.定位问题（你得知道问题出在哪里，要不然从哪里调优呢）
>     2.解决问题（这个没有基本的方法来处理，因为不同的问题处理的方式方法不一样，得从实践中不断的探索，如sql调优，配置优化，硬件升级等等） 

## 步骤1：查询是否开启了慢查询

```mysql
mysql> show variables like '%slow%';
+---------------------------+--------------------------------+
| Variable_name             | Value                          |
+---------------------------+--------------------------------+
| log_slow_admin_statements | OFF                            |
| log_slow_slave_statements | OFF                            |
| slow_launch_time          | 2                              |
| slow_query_log            | ON                             |
| slow_query_log_file       | /data/mysql/localhost-slow.log |
+---------------------------+--------------------------------+
5 rows in set (0.01 sec)

mysql> 
```

以MySQL5.7为例

我这里是开启了，没有开启的，直接`set global slow_query_log=on;`就ok了。 

```mysql
mysql> set global slow_query_log=on;
Query OK, 0 rows affected (0.05 sec)
```

## 步骤2：设置慢查询的时间限制

mysql默认的慢查询时间是10秒，可以设置成其它的时间。 

```mysql
mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.03 sec)

mysql> set long_query_time=1;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'long_query_time';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 1.000000 |
+-----------------+----------+
1 row in set (0.00 sec)

mysql> 
```

**set global 只是全局session生效，重启后失效,如果需要以上配置永久生效，需要在mysql.ini（linux my.cnf）中配置**

## 步骤3：查看慢查询

show status like 'slow_queries'; 
它会显示慢查询sql的数目，具体的sql就在上面的Log file日志中可以看到。 

```mysql
mysql> show status like 'slow_queries';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Slow_queries  | 0     |
+---------------+-------+
1 row in set (0.01 sec)

mysql> 
```

其它命令

show processlist： 查看哪些线程在运行； 

show open tables：查看哪些表在使用。

# 慢查询分析日志

改一下慢查询配置

```mysql
mysql> set long_query_time=0.1;
Query OK, 0 rows affected (0.05 sec)

mysql> 
```

执行几条慢的SQL

```mysql
mysql> select count(*) from users;
+----------+
| count(*) |
+----------+
|   100005 |
+----------+
1 row in set (0.28 sec)

mysql> select * from users;
...
...
100005 rows in set (1.41 sec)

mysql> 
```

```mysql
mysql> select count(*) from user_address_copy;
+----------+
| count(*) |
+----------+
|    30006 |
+----------+
1 row in set (0.08 sec)

mysql> select * from user_address_copy;
...
...
30006 rows in set (0.39 sec)

mysql> 
```

vim 打开慢查询记录的文件`slow_query_log_file`       | /data/mysql/localhost-slow.log

`vim /data/mysql/localhost-slow.log`

localhost-slow.log 内容如下：

```mysql
/software/mysql/bin/mysqld, Version: 5.7.24 (MySQL Community Server (GPL)). started with:
Tcp port: 3306  Unix socket: /software/mysql/mysql.sock
Time                 Id Command    Argument
# Time: 2018-12-08T03:08:23.877322Z
# User@Host: root[root] @ localhost []  Id:    24
# Query_time: 0.551358  Lock_time: 0.000514 Rows_sent: 1  Rows_examined: 100005
use test;
SET timestamp=1544238503;
select count(*) from users;
# Time: 2018-12-08T03:09:06.038256Z
# User@Host: root[root] @ localhost []  Id:    24
# Query_time: 1.401716  Lock_time: 0.000220 Rows_sent: 100005  Rows_examined: 100005
SET timestamp=1544238546;
select * from users;
# Time: 2018-12-08T03:12:03.207302Z
# User@Host: root[root] @ localhost []  Id:    24
# Query_time: 0.395499  Lock_time: 0.000378 Rows_sent: 30006  Rows_examined: 30006
SET timestamp=1544238723;
select * from user_address_copy;

```

`Time` ：日志记录的时间

`User@Host`：执行的用户及主机

`Query_time`：查询耗费时间 **Lock_time** 锁表时间 **Rows_sent** 发送给请求方的记录条数 **Rows_examined** 语句扫描的记录条数

`SET timestamp` 语句执行的时间点

`select ....` 执行的具体语句

## 慢查询日志分析工具

<!--参考https://www.olinux.org.cn/mysql/950.html-->

<!--参考https://www.centos.bz/2012/01/active-mysql-slow-log-mysqldumpslow/-->

分析慢查询日志是[性能调优](https://www.olinux.org.cn/tag/%e6%80%a7%e8%83%bd%e8%b0%83%e4%bc%98)中获取信息的主要方式之一。

如果slow log比较小，那么可以直接使用vi等文本编辑器或less、more命令打开。但如果slow log过大，载入慢查询日志将耗费大量时间，这个时候就要考虑使用其他工具来对慢查询进行分析了。

mysql的自带工具[mysqldumpslow](https://www.olinux.org.cn/tag/mysqldumpslow)，可以有效的帮助我们对slow log进行筛选和分析。

官方文档5.7版本地址：https://dev.mysql.com/doc/refman/5.7/en/mysqldumpslow.html
参看官方文档可以略去本文。

![MYSQLæ§è½è°ä¼ï¼å®æ¹mysqldumpslowåæMysql slow log](http://ww4.sinaimg.cn/large/006tNc79gy1g5xasl6slsj30c30e10te.jpg)

执行`mysqldumpslow –h`可以查看帮助信息。
主要介绍两个参数-s和-t
-s 这个是排序参数，可选的有：
​	al: 平均锁定时间
​	ar: 平均返回记录数
​	at: 平均查询时间
​	c: 计数
​	l: 锁定时间
​	r: 返回记录
​	t: 查询时间

-t n 显示头n条记录。

实例：
mysqldumpslow -s c -t 20 host-slow.log

mysqldumpslow -s r -t 20 host-slow.log

上述命令可以看出访问次数最多的20个sql语句和返回记录集最多的20个sql。

mysqldumpslow -t 10 -s t -g “left join” host-slow.log

这个是按照时间返回前10条里面含有左连接的sql语句。

用了这个工具就可以查询出来那些sql语句是性能的瓶颈，进行优化，比如加索引，该应用的实现方式等。

**其他工具**

//TODO 暂时还没了解，先记录一下

mysqlsla
pt-query-digest



参考链接：https://www.cnblogs.com/WONDERFUL-cnblogs/p/5481358.html