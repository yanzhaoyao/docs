# 1. 存储引擎介绍

> 1. 插拔式的插件方式
> 2. 存储引擎是指定在表之上的，即一个库中的每一个表都可以指定专用的存储引擎。
> 3. 不管表采用什么样的存储引擎，都会在数据区，产生对应的一个frm文件（表结构定义描述文件）

## 1.1 CSV存储引擎

### 1.1.1 mysql官网介绍[CVS链接](https://dev.mysql.com/doc/refman/5.7/en/csv-storage-engine.html)

## 1.1.2 特点：

不能定义索引、列定义必须为`NOT NULL`、不能设置`自增列`-->不适用大表或者数据的在线处理

`CSV`数据的存储用`,`隔开，可直接编辑`CSV`文件进行数据的编排-->数据安全性低

注：编辑之后，要生效使用flush table XXX 命令

### 1.1.3 应用场景

数据的快速导出导入

表格直接转换成CSV

## 1.2 Archive存储引擎

### 1.2.1 MySQL官网介绍[Archive链接](https://dev.mysql.com/doc/refman/5.7/en/archive-storage-engine.html)

The `ARCHIVE` storage engine produces special-purpose tables that store large amounts of unindexed data in a very small footprint.

（该`ARCHIVE`存储引擎生成专用表，这些表占用非常小的内存，能存储大量未索引的数据。）

官网介绍太多了，不贴了。

压缩协议进行数据的存储

数据存储为ARZ文件格式

### 1.2.2 特点

只支持insert和select两种操作

只允许自增ID列建立索引

行级锁

不支持事务

数据占用磁盘少

### 1.2.3 应用场景

日志系统

大量的设备数据采集

## 1.3 Memory存储引擎

### 1.3.1 MySQL官网Memory存储引擎[链接](https://dev.mysql.com/doc/refman/5.7/en/memory-storage-engine.html)

`MEMORY`存储引擎（以前称为 `HEAP`）创建具有存储在存储器中的内容的专用的表。由于数据容易受到崩溃，硬件问题或断电的影响，因此只能将这些表用作临时工作区或从其他表中提取数据的只读缓存。

数据都是存储在内存中，IO效率要比其他引擎高很多

服务重启数据丢失，内存数据表默认只有16M

### 1.3.2 特点

支持hash索引，B tree索引，默认hash（查找复杂度0(1)）

字段长度都是固定长度varchar(32)=char(32)

不支持大数据存储类型字段如 blog，text

表级锁

### 1.3.3 应用场景

等值查找热度较高数据

查询结果内存中的计算，大多数都是采用这种存储引擎

作为临时表存储需计算的数据

## 1.4 Myisam存储引擎

## 1.4.1 MySQL官网Myisam存储引擎[链接](https://dev.mysql.com/doc/refman/5.7/en/myisam-storage-engine.html)

Mysql5.5版本之前的默认存储引擎

较多的系统表也还是使用这个存储引擎

系统临时表也会用到Myisam存储引擎

每个`MyISAM`表都存储在三个文件的磁盘上。这些文件的名称以表名开头，并具有指示文件类型的扩展名。

`.frm` 文件存储表格式。

`.MYD`（`MYData`）数据文件。

`.MYI` （`MYIndex`）索引文件。

### 1.4.2 特点

> a，select count(*) from table 无需进行数据的扫描
>
> b，数据（MYD）和索引（MYI）分开存储
>
> c，表级锁
>
> d，不支持事务

贴一个文章[1分钟了解MyISAM与InnoDB的索引差异](https://mp.weixin.qq.com/s/FUXPXKfKyjxAvMUFHZm9UQ)

## 1.5 Innodb存储引擎

## 1.5.1 MySQL官网Innodb存储引擎[链接](https://dev.mysql.com/doc/refman/5.7/en/innodb-introduction.html)

Mysql5.5及以后版本的默认存储引擎
Key Advantages：

​	Its DML operations follow the ACID model [事务ACID]

​	Row-level locking[行级锁]

​	InnoDB tables arrange your data on disk to optimize queries

​	based on primary keys[聚集索引（主键索引）方式进行数据存储]

​	To maintain data integrity, InnoDB supports FOREIGN KEY

​	constraints[支持外键关系保证数据完整性]

# 2、对比

https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html

**Table  Storage Engines Feature Summary**

| Feature                                                      | MyISAM       | Memory           | InnoDB       | Archive | NDB          |
| ------------------------------------------------------------ | ------------ | ---------------- | ------------ | ------- | ------------ |
| B-tree indexes（B树索引）                                    | Yes          | Yes              | Yes          | No      | No           |
| Backup/point-in-time recovery (note 1)（备份/时间点恢复）    | Yes          | Yes              | Yes          | Yes     | Yes          |
| Cluster database support（群集数据库支持）                   | No           | No               | No           | No      | Yes          |
| **Clustered indexes**（**聚集索引**）                        | No           | No               | **Yes**      | No      | No           |
| Compressed data（压缩数据）                                  | Yes (note 2) | No               | Yes          | Yes     | No           |
| **Data caches**（**数据缓存**）                              | No           | N/A              | **Yes**      | No      | Yes          |
| Encrypted data(note 3)（加密数据）                           | Yes          | Yes              | Yes          | Yes     | Yes          |
| **Foreign key support**（**外键支持**）                      | No           | No               | **Yes**      | No      | Yes (note 4) |
| Full-text search indexes（全文搜索索引）                     | Yes          | No               | Yes (note 5) | No      | No           |
| Geospatial data type support（地理空间数据类型支持）         | Yes          | No               | Yes          | Yes     | Yes          |
| 地理空间索引支持                                             | Yes          | No               | Yes (note 6) | No      | No           |
| Hash indexes(哈希索引)                                       | No           | Yes              | No (note 7)  | No      | Yes          |
| Index caches(索引缓存)                                       | Yes          | N/A              | Yes          | No      | Yes          |
| **Locking granularity**（**锁定粒度**）                      | Table        | Table            | **Row**      | Row     | Row          |
| **MVCC**（Multi-Version Concurrency Control 多版本并发控制） | No           | No               | **Yes**      | No      | No           |
| Replication support (note 1)（复制支持）                     | Yes          | Limited (note 8) | Yes          | Yes     | Yes          |
| Storage limits （存储限制）                                  | 256TB        | RAM              | 64TB         | None    | 384EB        |
| T-tree indexes（T树索引）                                    | No           | No               | No           | No      | Yes          |
| **Transactions** (**事务**)                                  | No           | No               | **Yes**      | No      | Yes          |
| Update statistics for data dictionary(更新数据字典的统计信息) | Yes          | Yes              | Yes          | Yes     | Yes          |

**上表重点关注加粗部分**

**注：**

1.在服务器中实现，而不是在存储引擎中实现。

2.仅在使用压缩行格式时才支持压缩的MyISAM表。使用带MyISAM的压缩行格式的表是只读的。

3.通过加密功能在服务器中实现。MySQL 5.7及更高版本中提供了静态数据表空间加密。

4.MySQL Cluster NDB 7.3及更高版本支持外键。

5.在MySQL 5.6及更高版本中可以使用InnoDB对FULLTEXT索引的支持。

6.MySQL 5.7及更高版本中提供了InnoDB对地理空间索引的支持。

7.InnoDB在内部利用哈希索引来实现其自适应哈希索引功能。

8.请参阅本节后面的讨论。