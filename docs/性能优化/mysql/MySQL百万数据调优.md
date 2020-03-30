# MySQL百万数据调优

本文将记录一次SQL调优过程，sql比较简单，主要是项目中，持久层框架用到的是JPA

> 注：MySQL版本5.7.26

下图是项目DEBUG启动模式下的sql打印记录

![image-20191219214522555](/Users/xmly/Library/Application Support/typora-user-images/image-20191219214522555.png)

> 上图两段SQL，由于是jpa分页查询，一个是查询count，另一个则是limit分页查询
>
> jpa原来怎么坑，查询count竟然不是用select count(*) from table,而是列出来了所有的字段，太不友好

## 前置知识点

`EXPLAIN` 查询执行计划

![image-20191219230355928](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2fipvdyhj31a006sacp.jpg)

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

**执行计划-type`(重点、重点、重点)`**

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

## 1. 数据准备

### 1.1 表结构如下

```mysql
CREATE TABLE `base_winxuan_goods` (
  `ID` varchar(40) NOT NULL COMMENT '主键',
  `BOOKAUTHOR` varchar(600) DEFAULT NULL COMMENT '作者介绍',
  `BRAND_CODE` varchar(40) DEFAULT NULL COMMENT '发行商编号(出版社)',
  `CLASSIFICATION` varchar(200) DEFAULT NULL COMMENT '中图法分类',
  `DIRECTORY` varchar(40) DEFAULT NULL COMMENT '目录',
  `GOODS_CODE` varchar(80) NOT NULL COMMENT '供应商商品ID号',
  `GOODS_COPYRIGHT` varchar(20) DEFAULT NULL COMMENT '版次',
  `GOODS_FORMAT` varchar(40) DEFAULT NULL COMMENT '装帧',
  `GOODS_IS_NEW` decimal(1,0) DEFAULT NULL COMMENT '是否是新品标识：1-新品，2-非新品',
  `GOODS_ISBN` varchar(80) DEFAULT NULL COMMENT 'isbn',
  `GOODS_NAME` varchar(500) DEFAULT NULL COMMENT '商品名称',
  `GOODS_NATION` varchar(40) DEFAULT NULL COMMENT '国别',
  `GOODS_PACKAGE` varchar(40) DEFAULT NULL COMMENT '包装',
  `GOODS_PAGE` decimal(10,0) DEFAULT NULL COMMENT '页码',
  `GOODS_PAPER` varchar(40) DEFAULT NULL COMMENT '用纸',
  `GOODS_PRESS_HOUSE` varchar(40) DEFAULT NULL COMMENT '出版社名称',
  `GOODS_PRICE` decimal(16,6) DEFAULT NULL COMMENT '定价',
  `GOODS_PRINT_DATE` datetime DEFAULT NULL COMMENT '印刷时间',
  `GOODS_PUBLISH_DATE` datetime DEFAULT NULL COMMENT '出版时间',
  `GOODS_PUT_AWAY` decimal(1,0) DEFAULT NULL COMMENT '商品上下架 1 上架 2 下架',
  `GOODS_SUIT` decimal(1,0) DEFAULT NULL COMMENT '是否套装',
  `GOODS_SUPPLIER` varchar(80) DEFAULT NULL COMMENT '供应商编号',
  `GOODS_TRANSLATION` decimal(1,0) DEFAULT NULL COMMENT '订购限制 1 正常 2 预售 3 暂时禁止订货',
  `GOODS_WORDS` varchar(40) DEFAULT NULL COMMENT '字数',
  `GOODS_WRITER` varchar(600) DEFAULT NULL COMMENT '作者',
  `PIC_ARR` varchar(4000) DEFAULT NULL COMMENT '图片',
  `SITE_CODE` varchar(40) DEFAULT NULL COMMENT '商品分类（末级）',
  `SUMMARY` varchar(500) DEFAULT NULL COMMENT '内容提要',
  `TAG_CODE` varchar(500) DEFAULT NULL COMMENT '标签(多个标签编号逗号分隔，)',
  `TJ_CATE_GORIES` varchar(500) DEFAULT NULL,
  `UPDATE_TIME` datetime DEFAULT NULL COMMENT '更新时间',
  `YUNHANID` varchar(50) DEFAULT NULL COMMENT '商品平台编号',
  `DR` decimal(1,0) DEFAULT NULL COMMENT '删除标记',
  `TS` datetime DEFAULT NULL COMMENT '时间戳',
  `CREATOR` varchar(40) DEFAULT NULL COMMENT '创建人',
  `CREATION_TIME` datetime DEFAULT NULL COMMENT '创建时间',
  `MODIFIER` varchar(40) DEFAULT NULL COMMENT '修改人',
  `MODIFIED_TIME` datetime DEFAULT NULL COMMENT '修改时间',
  PRIMARY KEY (`ID`) USING BTREE,
  UNIQUE KEY `goodsCodeIndex` (`GOODS_CODE`) USING BTREE,
  KEY `goodsIsbnIndex` (`GOODS_ISBN`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品';
```

### 1.2 可以注意到除了主键以外，已经有了两个索引

```mysql
UNIQUE KEY `goodsCodeIndex` (`GOODS_CODE`) USING BTREE,
KEY `goodsIsbnIndex` (`GOODS_ISBN`) USING BTREE
```

### 1.3 数据量(1342074)

(数据的话就不提供了)

![image-20191219223528267](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2ep4edo7j313s07eq4r.jpg)

### 1.4 我们主要优化分页查询的，SQL如下：

不知道为啥，jpa打印的SQL，出现两个dr=0的查询条件（逻辑删除），我们后面会去掉一个dr查询

```mysql
SELECT DISTINCT
	winxuangoo0_.id AS id1_89_,
	winxuangoo0_.creation_time AS creation2_89_,
	winxuangoo0_.creator AS creator3_89_,
	winxuangoo0_.dr AS dr4_89_,
	winxuangoo0_.modified_time AS modified5_89_,
	winxuangoo0_.modifier AS modifier6_89_,
	winxuangoo0_.ts AS ts7_89_,
	winxuangoo0_.bookauthor AS bookauth8_89_,
	winxuangoo0_.brand_code AS brand_co9_89_,
	winxuangoo0_.classification AS classif10_89_,
	winxuangoo0_.DIRECTORY AS directo11_89_,
	winxuangoo0_.goods_code AS goods_c12_89_,
	winxuangoo0_.goods_copyright AS goods_c13_89_,
	winxuangoo0_.goods_format AS goods_f14_89_,
	winxuangoo0_.goods_is_new AS goods_i15_89_,
	winxuangoo0_.goods_isbn AS goods_i16_89_,
	winxuangoo0_.goods_name AS goods_n17_89_,
	winxuangoo0_.goods_nation AS goods_n18_89_,
	winxuangoo0_.goods_package AS goods_p19_89_,
	winxuangoo0_.goods_page AS goods_p20_89_,
	winxuangoo0_.goods_paper AS goods_p21_89_,
	winxuangoo0_.goods_press_house AS goods_p22_89_,
	winxuangoo0_.goods_price AS goods_p23_89_,
	winxuangoo0_.goods_print_date AS goods_p24_89_,
	winxuangoo0_.goods_publish_date AS goods_p25_89_,
	winxuangoo0_.goods_put_away AS goods_p26_89_,
	winxuangoo0_.goods_suit AS goods_s27_89_,
	winxuangoo0_.goods_supplier AS goods_s28_89_,
	winxuangoo0_.goods_translation AS goods_t29_89_,
	winxuangoo0_.goods_words AS goods_w30_89_,
	winxuangoo0_.goods_writer AS goods_w31_89_,
	winxuangoo0_.pic_arr AS pic_arr32_89_,
	winxuangoo0_.site_code AS site_co38_89_,
	winxuangoo0_.summary AS summary33_89_,
	winxuangoo0_.tag_code AS tag_cod34_89_,
	winxuangoo0_.tj_cate_gories AS tj_cate35_89_,
	winxuangoo0_.update_time AS update_36_89_,
	winxuangoo0_.yunhanid AS yunhani37_89_ 
FROM
	base_winxuan_goods winxuangoo0_ 
WHERE
	winxuangoo0_.dr = 0 
	AND winxuangoo0_.dr = 0 
ORDER BY
	winxuangoo0_.creation_time DESC 
```

## 2.开始优化

### 2.1 EXPLAIN SQL

```mysql
EXPLAIN SELECT DISTINCT *** 省略 ***
```

![image-20191219222334980](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2ecsns56j31lk0u0qky.jpg)

`type=ALL`，结果是全表扫描，`Using filesort` 还用到了外部文件排序。

第一个想到是不是给dr加索引呢？当然肯定不是，dr数据的离散性太差了（只有0和1）。我们可以试一下

### 2.2 给dr添加单列索引后EXPLAIN

```mysql
# 添加INDEX(普通索引) 
ALTER TABLE `base_winxuan_goods` ADD INDEX drIndex ( `dr` ) ;
```

```mysql
EXPLAIN *** 省略 ***
```

![image-20191219223208170](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2elqb0zmj31ju0u07m5.jpg)

虽然`type=ref` ，但仍然回表扫描了，扫描了几乎一半的数据量，且用到了外部文件排序`Using filesort`

此种方式不推荐，把**dr的索引删掉**，我们继续验证

### 2.3 给creation_time字段加上索引

```mysql
# 添加INDEX(普通索引) 
ALTER TABLE `base_winxuan_goods` ADD INDEX creationTimeIndex ( `creation_time` ) ;
```

#### 2.3.1 加索引前`EXPLAIN`

![image-20191219223833463](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2esdi1coj31jm0can2f.jpg)

#### 2.3.2 加索引后`EXPLAIN`

![image-20191219224039543](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2euj8604j31la0ckq85.jpg)

可以看见，加上`creationTimeIndex`索引后，`type=index`，没有使用外部文件排序了。效果很好是不是？

有没有可以在优化的可能性？



可以做个延迟查询。先查id在回表查询（这里利用覆盖索引，单查id可以不用回表，获取到id后再用id回表查询），我们继续

那么问题来了，什么是延迟查询？

如：`select .. from where id in ( select id from where ... limit`

> 可惜项目中的主键都不是自增的ID（用的UUID），否则ORDER BY ID非常舒服

延迟加载 用`in` 或者 `join`都可以

我们先用`in`做个延迟查询

### 2.4 用`in`做个延迟查询

> 注：creation_time字段的索引不要删除

![image-20191219225008894](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2f4hi8vlj31sj0u07p0.jpg)

结果：`Using temporary`使用了临时表，且`type=ALL`，`rows=1195655`,也基本上扫描了快全表的数据量了，效果不好，我们继续。

### 2.5 用join做个延迟查询

#### 2.5.1 LEFT JOIN

![image-20191219225222828](https://tva1.sinaimg.cn/large/006tNbRwly1ga2f6tpkgoj31uc0rah3k.jpg)

结果: 和IN条件差不多，可能还没有IN的好

#### 2.5.2 INNER JOIN

![image-20191219225644748](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2fba6xo4j31ik0na4ce.jpg)

结果：可以看到`rows=20`,只要扫描了limit的数量，目前是优化效果最好的

## 3. limit 去掉直接查全库

![image-20191219225917396](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2fdze2xoj31qy0mytmm.jpg)

## 4. EXPLAIN 去掉 直接查全库

