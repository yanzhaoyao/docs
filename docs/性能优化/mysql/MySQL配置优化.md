# MySQL配置优化

​	如何寻找MySQL配置文件，MySQL内存参数如何配置

# MySQL服务器参数类型

基于参数的作用域：

全局参数

```mysql
set global autocommit = ON/OFF;
```

会话参数(会话参数不单独设置则会采用全局参数)

```mysql
set session autocommit = ON/OFF;
```


注意：

- 全局参数的设定对于已经存在的会话无法生效
- 会话参数的设定随着会话的销毁而失效
- 全局类的统一配置建议配置在默认配置文件中，否则重启服务会导致配置失效

# 寻找配置文件不迷路

假如刚进公司，boss丢了一个服务给你，让你去优化一下MySQL的配置，只需要执行命令`mysql --help`

mysql --help 寻找配置文件的位置和加载顺序

```mysql
[root@localhost mysql]# mysql --help | grep -A 1 'Default options are read from the following files in the given order'
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ~/.my.cnf 
[root@localhost mysql]# 
```

当然，如果记不住怎么多，只需要记住`mysql --help` 即可。

# 常见的全局配置文件配置

```mysql
port = 3306
socket = /tmp/mysql.sock
basedir = /usr/local/mysql
datadir = /data/mysql
pid-file = /data/mysql/mysql.pid
user = mysql
bind-address = 0.0.0.0
max_connections=2000
lower_case_table_names = 0 #表名区分大小写
server-id = 1
tmp_table_size=16M
transaction_isolation = REPEATABLE-READ
ready_only=1
```

# MySQL内存参数配置

每一个connection内存参数配置：
`sort_buffer_size` connection排序缓冲区大小
​	建议256K(默认值)-> 2M之内（**若配置为2M**）
​	当查询语句中有需要文件排序功能时，马上为connection分配配置的内存大小(**2M**)

`join_buffer_size` connection关联查询缓冲区大小
​	建议256K(默认值)-> 1M之内
​	当查询语句中有关联查询时，马上分配配置大小的内存用这个关联查询，所以有可能在一个查询语句中会分配很多个关联查询缓冲区

上述配置4000连接占用内存：

**4000*((256k/1024)M + (256K/1024)M) ~= 2G**

`Innodb_buffer_pool_size` Innodb buffer/cache的大小（默认128M）

Innodb_buffer_pool
​	数据缓存
​	索引缓存
​	缓冲数据
​	内部结构

大的缓冲池可以减小多次磁盘I/O访问相同的表数据以提高性能
参考计算公式：
**Innodb_buffer_pool_size = （总物理内存 - 系统运行所用 - connection 所用）* 90%**

`wait_timeout` 服务器关闭非交互连接之前等待活动的秒数

`innodb_open_files`限制Innodb能打开的表的个数

`innodb_write_io_threads`
`innodb_read_io_threads`
​	innodb使用后台线程处理innodb缓冲区数据页上的读写 I/O(输入输出)请求

`innodb_lock_wait_timeout`
​	InnoDB事务在被回滚之前可以等待一个锁定的超时秒数

https://www.cnblogs.com/wyy123/p/6092976.html 常见配置的帖子