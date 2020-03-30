# Mycat分库分表实战案例demo

## 环境准备

两台虚拟机

Host1 : 192.168.12.66

Host2 : 192.168.12.88

MySQL 5.7

Mycat 1.6

![121723051595077](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc2s0pahj30zv0q4dk2.jpg)

## 分库分表规则

```
mycat定义两个逻辑库，分别为db_user、db_store

表db_user按照模分成2个片,分别落在host1、host2上

表db_store为主从复制模式，分别落在host1、host2上
```

创建数据库和表结构

1. 分别在HOST1、HOST2执行db_user.sql

```mysql
CREATE DATABASE db_user DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;

use db_user;

-- ----------------------------
-- Table structure for data_dictionary
-- ----------------------------
DROP TABLE IF EXISTS `data_dictionary`;
CREATE TABLE `data_dictionary` (
  `dataDictionaryID` int(11) NOT NULL COMMENT '数据字典ID',
  `displayName` varchar(32) COLLATE utf8_bin DEFAULT NULL COMMENT '显示名称',
  `value` varchar(255) COLLATE utf8_bin DEFAULT NULL COMMENT '数据字典取值',
  `createTime` datetime DEFAULT NULL COMMENT '创建时间',
  `lastUpdate` datetime DEFAULT NULL COMMENT '最后更新时间',
  PRIMARY KEY (`dataDictionaryID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

-- ----------------------------
-- Table structure for user_address
-- ----------------------------
DROP TABLE IF EXISTS `user_address`;

CREATE TABLE `user_address` (
  `addressID` int(11) NOT NULL COMMENT '地址ID',
  `receiver` varchar(16) COLLATE utf8_bin DEFAULT NULL COMMENT '收货人',
  `addressDetail` varchar(32) COLLATE utf8_bin DEFAULT NULL COMMENT '地址详细',
  `userID` int(11) NOT NULL COMMENT '用户ID',
  `createTime` datetime DEFAULT NULL COMMENT '创建时间',
  `lastUpdate` datetime DEFAULT NULL COMMENT '最后更新时间',
  PRIMARY KEY (`addressID`,`userID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

-- ----------------------------
-- Table structure for users
-- ----------------------------
DROP TABLE IF EXISTS `users`;
CREATE TABLE `users` (
  `userID` int(11) NOT NULL COMMENT '用户ID',
  `username` varchar(16) COLLATE utf8_bin DEFAULT NULL COMMENT '用户名',
  `phoneNum` varchar(32) COLLATE utf8_bin DEFAULT NULL COMMENT '手机号码',
  `age` int(11) DEFAULT NULL COMMENT '年龄',
  `ddID` int(11) DEFAULT NULL COMMENT '所属会员类型',
  `createTime` datetime DEFAULT NULL COMMENT '注册时间',
  `lastUpdate` datetime DEFAULT NULL COMMENT '最后更新时间',
  PRIMARY KEY (`userID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

1. 只需要在HOST1(192.168.12.66)上执行db_store.sql，HOST2主从复制会自动同步数据

```mysql
CREATE DATABASE  db_store DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;

use db_store;
-- ----------------------------
-- Table structure for employee
-- ----------------------------
DROP TABLE IF EXISTS `employee`;
CREATE TABLE `employee` (
  `employeeID` int(11) NOT NULL,
  `userName` varchar(16) COLLATE utf8_bin DEFAULT NULL,
  `phoneNum` varchar(32) COLLATE utf8_bin DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `createTime` datetime DEFAULT NULL,
  `lastUpdate` datetime DEFAULT NULL,
  PRIMARY KEY (`employeeID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

-- ----------------------------
-- Table structure for store
-- ----------------------------
DROP TABLE IF EXISTS `store`;
CREATE TABLE `store` (
  `storeID` int(11) NOT NULL,
  `storeName` varchar(32) COLLATE utf8_bin DEFAULT NULL,
  `storeAddress` varchar(256) COLLATE utf8_bin DEFAULT NULL,
  `createTime` datetime DEFAULT NULL,
  `lastUpdate` datetime DEFAULT NULL,
  PRIMARY KEY (`storeID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

执行完毕后如下

![1545309851511](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc39exsqj305j091aa2.jpg)

## 配置conf文件

两台host主机Mycat的conf目录下配置三个配置文件

schema.xml

server.xml

rule.xml

## 重启mycat

HOST1(192.168.12.66)、HOST2（192.168.12.88）

```mysql
[root@localhost bin]# pwd
/software/mycat/bin
[root@localhost bin]# ./mycat restart
Stopping Mycat-server...
Mycat-server was not running.
Starting Mycat-server...
[root@localhost bin]# ./mycat status
Mycat-server is running (8228).
```

注意mycat是java编写的，因此需要安装jdk,此处省略

## 连接mycat

linux 控制台 `mysql -uroot -padmin -P8066 -h192.168.12.66`

查看逻辑库`show database;`

```mysql
[root@localhost bin]# mysql -uroot -padmin -P8066 -h192.168.12.66
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.6.29-mycat-1.6-RELEASE-20161028204710 MyCat Server (monitor)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show database;
+----------+
| DATABASE |
+----------+
| db_store |
| db_user  |
+----------+
2 rows in set (0.01 sec)

mysql> 
```

只能看到逻辑库 `db_store`、`db_user`

**我们用Navicat连接如下**

注意8066端口要开放，我这边直接关闭了防火墙。

![1545312790521](http://ww1.sinaimg.cn/large/006tNc79gy1g5xc3j3xnrj30di0f9aab.jpg)

![1545312626064](http://ww3.sinaimg.cn/large/006tNc79gy1g5xc3q1lgij305o09umx7.jpg)



## 插入数据测试

### 分片表users

通过mycat连接db_user库，执行如下SQL

insertdb_user_users.sql

```mysql
INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('1', '张1', '13611111111', '31', '2', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('2', '王二', '13622222222', '32', '5', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('3', '李三', '13633333333', '33', '3', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('4', '赵四', '13644444444', '34', '1', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('5', '田五', '13655555555', '35', '3', '2018-10-10 13:39:41', '2018-10-10 13:39:41');

```

如下执行

```mysql
mysql> show databases;
+----------+
| DATABASE |
+----------+
| db_store |
| db_user  |
+----------+
2 rows in set (0.01 sec)

mysql> use db_user;
Database changed
mysql> show tables;
+-------------------+
| Tables in db_user |
+-------------------+
| data_dictionary   |
| users             |
| user_address      |
+-------------------+
3 rows in set (0.00 sec)

mysql> select * from users;
Empty set (0.14 sec)

mysql> 
mysql> INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('1', '张1', '13611111111', '31', '2', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.40 sec)

mysql> INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('2', '王二', '13622222222', '32', '5', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.03 sec)

mysql> INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('3', '李三', '13633333333', '33', '3', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.04 sec)

mysql> INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('4', '赵四', '13644444444', '34', '1', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.09 sec)

mysql> INSERT INTO `users`(userID,userName,phoneNum,age,ddID,createTime,lastUpdate) VALUES ('5', '田五', '13655555555', '35', '3', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.00 sec)
```

用Navicat查看数据

Mycat的users数据

![1545313275374](http://ww1.sinaimg.cn/large/006tNc79gy1g5xc52z84ij30nn05wjrv.jpg)

再看host1和host2上面的数据users数据

66Master

![1545313400976](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc5465m1j30lx0503yt.jpg)

88Slave

![1545313443778](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc56j167j30p4054glw.jpg)

可以看到，已经按照我们制定的取模规则进行分片存储数据了。

### ER表user_address

#### Mycat 上执行

`insertdb_user_user_address.sql`

```
INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('1', '张一', '深圳南山科技园', '1', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('2', '张一', '深圳龙华地铁站', '1', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('3', '王二', '长沙麓谷软件园', '2', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('4', '赵四', '长沙麓谷企业广场', '4', '2018-10-10 13:46:36', '2018-10-10 13:46:36');
INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('5', '李三', '深圳福田侨香', '3', '2018-10-10 13:46:36', '2018-10-10 13:46:36');
```

执行结果

```mysql
mysql> INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('1', '张一', '深圳南山科技园', '1', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.03 sec)

mysql> INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('2', '张一', '深圳龙华地铁站', '1', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.02 sec)

mysql> INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('3', '王二', '长沙麓谷软件园', '2', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.03 sec)

mysql> INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('4', '赵四', '长沙麓谷企业广场', '4', '2018-10-10 13:46:36', '2018-10-10 13:46:36');
Query OK, 1 row affected (0.02 sec)

mysql> INSERT INTO `user_address`(addressID,receiver,addressDetail,userID,createTime,lastUPdate) VALUE ('5', '李三', '深圳福田侨香', '3', '2018-10-10 13:46:36', '2018-10-10 13:46:36');
Query OK, 1 row affected (0.01 sec)
```

#### 用Navicat查看

##### Mycat上user_address数据

![1545313809580](http://ww3.sinaimg.cn/large/006tNc79gy1g5xc59d00aj30lm07tdgg.jpg)



##### 66Master上user_address数据

![1545313900367](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc5ba7q5j30ok06igm2.jpg)



##### 88Slave上user_address数据

![1545313925773](http://ww4.sinaimg.cn/large/006tNc79gy1g5xc5d7zgpj30on0603yw.jpg)

### 全局表data_dictionary

#### mycat上插入数据

`insertdb_user_data_dictionary.sql`

```
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('1', '白银', 'BY', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('2', '黄金', 'HJ', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('3', '砖石', 'ZS', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('4', '大师', 'DS', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('5', '王者', 'WZ', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
```

执行结果

```mysql
mysql> INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('1', '白银', 'BY', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
 ('2', '黄金', 'HJ', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('3', '砖石', 'ZS', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('4', '大师', 'DS', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('5', '王者', 'WZ', Query OK, 1 row affected (0.02 sec)

mysql> INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('2', '黄金', 'HJ', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.05 sec)

mysql> INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('3', '砖石', 'ZS', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('4', '大师', 'DS', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO `data_dictionary`(dataDictionaryID,displayName,value,createTime,lastUpdate) VALUE ('5', '王者', 'WZ', '2018-10-10 13:39:41', '2018-10-10 13:39:41');
Query OK, 1 row affected (0.02 sec)
```

#### Navicat查看数据

##### 66Mycat上查看

![1545314173497](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc5gnpv7j30ls06a0t7.jpg)



##### 66Master上查看data_dictionary

![1545314266321](http://ww4.sinaimg.cn/large/006tNc79gy1g5xc5iuvcwj30n6067gm2.jpg)



##### 88Slave上查看data_dictionary

![1545314294009](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc5k6nytj30lq0600t7.jpg)

全局表每个节点的数据都一样

### db_store测试

#### insert_db_store_store.sql

```mysql
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (1, '深圳宝安区圣淘沙店', '宝安区圣淘沙骏园5栋18号商铺', '20170413101010', '20170413101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (2, '深圳罗湖区红宝路1店', '罗湖区红宝路112号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (3, '深圳福田区梅华路店', '福田区上梅林梅华路上梅林菜场斜对面', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (4, '深圳福田区景田店', '福田区景田西住宅区七栋首层(民润超对面)', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (5, '深圳宝安区富通苑店', '宝安区新安街道48区富通苑A栋115号', '20170413101010', '20170413101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (6, '深圳罗湖区海富花园店', '罗湖区深南东路海富花园福怡阁底层A8铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (7, '深圳龙岗区四季花城店', '龙岗区坂田四季花城B19号商铺', '20180131101010', '20180131101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (8, '深圳南山区蔚蓝海岸店', '南山区登良路蔚蓝海岸商铺B09号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (9, '深圳福田区中康路店', '福田区上梅林中康路梅林医院斜对面振业梅苑B栋110号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (10, '深圳罗湖区松泉山庄店', '罗湖区太白路松泉山庄4栋6栋楼裙碧涟阁首层商场103-3号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (11, '深圳福田区沙嘴店', '福田区沙嘴村1坊96号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (12, '深圳宝安区桃源居3店', '宝安区前进路桃源居48区', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (13, '深圳南山区学府店', '南山区后海大道西宏观苑商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (14, '深圳福田区车公庙店', '福田区泰然工贸园四路105栋首层(泰康轩旁)', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (15, '深圳福田区金地店', '福田区金地一路金海丽名居102号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (16, '深圳罗湖区龙园山庄店', '罗湖区龙园山庄1栋102A', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (17, '深圳福田区天然居店', '福田区翔名苑B—8A商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (18, '深圳宝安区三联路店', '宝安区龙华镇三联路181号一楼右层', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (19, '深圳宝安区深业新岸线店', '宝安区深业新岸线2栋11号铺', '20170413101010', '20170413101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (20, '深圳宝安区风和日丽店', '宝安区龙华镇丰润花园13栋21号铺', '20171016101010', '20171016101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (21, '深圳龙岗区茂盛路店', '龙岗区横岗镇茂盛路16号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (22, '深圳南山区西海湾店', '南山区南商路西海湾花园单身公寓A4号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (23, '深圳南山区招商海月店', '南山区后海路招商海月花园24栋1-1', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (24, '深圳罗湖区布心店', '罗湖区布心村太白路1号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (25, '深圳宝安区美丽365花园店', '宝安区龙华镇东环一路美丽365花园B2栋102商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (26, '深圳南山区阳光棕榈园店', '南山区阳光棕榈园商业街九栋105室', '20180404101010', '20180404101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (27, '深圳宝安区天骄店', '宝安区天骄世家花园125#商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (28, '深圳宝安区东源阁店', '宝安区龙华镇东环二路东源阁C区一栋112商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (29, '深圳福田区中信广场店', '福田区同心路铺尾村54栋103号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (30, '深圳福田区碧海云天店', '福田白石洲路以北红树东方家园15栋一层商场10#', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (31, '深圳宝安区建安新村店', '宝安区西乡镇上川路建安新村104号铺', '20170413101010', '20170413101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (32, '深圳福田区益田村店', '福田区益田路1005号益田村(高层区)108栋1层105A号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (33, '新安湖店(停用)', NULL, '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (34, '深圳南山区南山市场店', '南山区南新路2025号之二', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (35, '上沙东村店(停用)', '福田区农轩路农科中心', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (36, '深圳宝安区桃源居1店', '宝安区前进路桃源居4区1栋117号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (37, '深圳福田区梅兴苑店', '福田区梅华路梅兴苑四栋13号铺', '20180404101010', '20180404101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (38, '深圳南山区缤纷店(停)', '深圳市南山区缤纷假日广场125号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (39, '深圳福田区莲花一村店', '福田区莲花一村17栋附一楼', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (40, '深圳宝安区富通天骏店', '宝安区龙华镇和平路富通天骏12号', '20171016101010', '20171016101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (41, '深圳罗湖区蔡屋围店', '罗湖区书城路蔡屋围新八坊41栋南', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (42, '深圳宝安区新境园店', '宝安区民治街梅龙南路东侧阳光新境园三栋一层2号', '20171016101010', '20171016101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (43, '深圳南山区育德佳园店(停用)', '南山区后海路育德佳园2栋10号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (44, '深圳福田区新洲一街店', '福田区新洲北村32号一楼南铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (45, '深圳宝安区缤纷世界店', '宝安区兴业路富通城一期B2栋125号商铺', '20170413101010', '20170413101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (46, '深圳龙岗区百合山庄店', '龙岗区布吉日尾坑百合山庄百薇苑142号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (47, '深圳宝安区景龙新村店', '宝安区龙华镇宝华路景龙新村', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (48, '深圳宝安区榕苑店', '宝安区龙华镇民治路榕苑花园', '20180404101010', '20180404101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (49, '深圳南山区明星店', '南山区前海路星海名城七组团4栋41号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (50, '深圳福田区福源店', '福田区彩田路福源大厦2014-15号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (51, '深圳龙岗区长龙路店', '龙岗区布吉长龙路113号B门面', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (52, '黄贝岭店(停)', '黄贝岭下村310栋', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (53, '深圳福田区梅林一村店', '福田区下梅林围面村1-2号102', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (54, '香榭里店(停用)', '福田区农轩路农科中心', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (55, '皇御苑店(停)', '福田区皇御苑B区7栋至11栋裙楼', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (56, '深圳福田区田面店', '福田区田面花园西侧', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (57, '深圳罗湖区松园店', '罗湖区松园路2号楼3号铺(红岭小学对面)', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (58, '皇岗新村店(停用)', '福田区皇岗新村25号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (59, '深圳宝安区水斗富豪店', '宝安区龙华镇水斗富豪新村二巷8号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (60, '深圳宝安区皇庭世纪店(停用)', '福田区皇岗村吉龙二村65号102铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (61, '深圳南山区海滨店', '南山区高新南路11道彩虹之岸商铺112号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (62, '深圳宝安区西城上筑店', '宝安区西城上筑花园1栋1-119号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (63, '深圳宝安区天悦龙庭店', '宝安区天悦龙庭B栋128号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (64, '深圳龙岗区康桥花园店', '龙岗区布吉镇中城康桥花园（二期）青之源17栋107', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (65, '深圳福田区宝田苑店', '福田区福强路宝田苑5B商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (66, '深圳龙岗区可园1店', '龙岗区可园社区13栋107铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (67, '深圳罗湖区渔民村店', '罗湖区渔民村商铺6C', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (68, '深圳福田区紫竹四路店', '福田区竹子林育星苑一栋首层C商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (69, '深圳宝安区民乐花园店', '宝安区民乐花园19栋103号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (70, '深圳罗湖区松泉公寓店', '罗湖区太白路松泉公寓17栋1楼A011号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (71, '深圳罗湖区田心村店', '罗湖区宝岗路田心村田心大厦1楼4号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (72, '潜龙鑫茂店(停)', '龙华潜龙鑫茂花园', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (73, '深圳龙岗区布吉莲花1店', '龙岗区布吉镇布吉莲花100号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (74, '深圳南山区现代城店', '南山区南光路现代华庭6栋1单元104号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (75, '金色花园店', '福田区莲花路金色家园３期裙楼Ａ02B', '20180811101010', '20180811101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (76, '深圳福田区翰岭苑店', '福田区翰岭花园1-2栋裙楼113号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (77, '深圳福田区百花三路店', '福田区百花三路南天2栋106B商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (78, '滨海之窗店(停)', NULL, '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (79, '深圳宝安区滢水山庄店', '宝安区滢水山庄二区8栋106', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (80, '深圳罗湖区莲塘国威店', '罗湖区莲塘国威路聚宝路182号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (81, '深圳罗湖区宝安南路店', '罗湖区宝安南2015号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (82, '百乐汇店中店', NULL, '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (83, '深圳龙岗区翠枫豪园店', '龙岗区布吉翠峰豪园1005号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (84, '深圳宝安区富士康1店', '宝安区龙华油松第十工业区东环二路二号富士康L1区商业街1楼第9号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (85, '深圳福田区新洲二街店', '福田区新洲村新洲二街绿景新苑10号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (86, '深圳宝安区锦绣江南店', '宝安区民治街道锦绣江南B1148号商铺', '20171016101010', '20171016101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (87, '测试门店1', NULL, '20180926101010', '20180926101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (88, '深圳宝安区观澜格澜郡店', '宝安区观澜街道观澜郡一期A02号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (89, '深圳龙岗区东大街1店', '龙岗区布吉东大街中翠花园111铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (90, '深圳福田区丽阳天下店', '福田区石厦路缔馨园裙楼111号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (91, '深圳宝安区桃源居2店(停)', '深圳市宝安区前进路桃源居118号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (92, '深圳福田区沙嘴2店(停)', '福田区', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (93, '深圳龙岗区丽湖花园店', '深圳市龙岗区丽湖花园湖彩阁9号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (94, '深圳宝安区龙泉花园店', '宝安区龙华龙泉花园A14号铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (95, '深圳罗湖区中兴路店', '罗湖区中兴路103号商铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (96, '深圳龙岗区东大街2店', '龙岗区布吉东大街桂芳园7区29栋D102铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (97, '深圳罗湖区莲塘聚福路店', '罗湖区莲塘聚福路鹏兴花园56栋114铺', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (98, '深圳南山区东滨路店', '南山区东滨路蛇口兰园大厦15A', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (99, '深圳福田区国都高尔夫花园店', '福田区新沙路国都高尔夫花园翠逸阁1层107C号', '20170330101010', '20170330101010');
INSERT INTO `store` (`storeID`, `storeName`, `storeAddress`, `createTime`, `lastUpdate`) VALUES (100, '东莞市中信凯旋城', '东莞市东城区火炼树怡丰都市广场怡祥阁B2铺', '20180223180232', '20180223180232');
```

执行不贴了

Navicat查看数据

Mycat上

![1545314824525](http://ww1.sinaimg.cn/large/006tNc79gy1g5xc5qrfq6j310n0htmzu.jpg)

66Master上

![1545314881873](http://ww1.sinaimg.cn/large/006tNc79gy1g5xc5t544hj30r40iv414.jpg)

88Slave上

![1545314940402](http://ww1.sinaimg.cn/large/006tNc79gy1g5xc5uird4j30r40ivacq.jpg)

#### insertdb_store_employee.sql

```mysql
insert employee (employeeID,userName,phoneNum,age,createTime,lastUpdate ) value (1,"张三","13611111111",21,'2018-10-10 11:31:23','2018-10-10 11:31:23'); 
insert employee (employeeID,userName,phoneNum,age,createTime,lastUpdate ) value (2,"李四","13622222222",22,'2018-10-10 11:31:23','2018-10-10 11:31:23'); 
insert employee (employeeID,userName,phoneNum,age,createTime,lastUpdate ) value (3,"王五","13633333333",23,'2018-10-10 11:31:23','2018-10-10 11:31:23'); 
insert employee (employeeID,userName,phoneNum,age,createTime,lastUpdate ) value (4,"赵六","13644444444",24,'2018-10-10 11:31:23','2018-10-10 11:31:23'); 
insert employee (employeeID,userName,phoneNum,age,createTime,lastUpdate ) value (5,"田七","13655555555",25,'2018-10-10 11:31:23','2018-10-10 11:31:23'); 
```

Navicat查看数据

66Mycat上

![1545315220642](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc5z301rj30ox07xq3g.jpg)

66Master上

![1545315169868](http://ww3.sinaimg.cn/large/006tNc79gy1g5xc605320j30om06y0t9.jpg)

88Slave上

![1545315139723](http://ww4.sinaimg.cn/large/006tNc79gy1g5xc62gl4xj30lf07cq3g.jpg)

