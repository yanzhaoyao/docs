# 一、安装前的检查

## 1.1 检查 linux 系统版本

```shell
# cat /etc/system-release
```

说明：小生的版本为 linux 64位：CentOS release 6.10 (Final)

## 1.2 检查是否安装了 mysql

```shell
# rpm -qa | grep mysql
```

若存在 mysql 安装文件，则会显示 mysql安装的版本信息

如：mysql-connector-odbc-5.2.5-6.el7.x86_64

卸载已安装的MySQL，卸载mysql命令，如下：

```shell
# rpm -e --nodeps mysql-connector-odbc-5.2.5-6.el7.x86_64
```

将/var/lib/mysql文件夹下的所有文件都删除干净。

> 细节注意：
>
> 　　检查一下系统是否存在 mariadb 数据库，如果有，一定要卸载掉，否则可能与 mysql 产生冲突。
>
> 　　系统安装模式的是最小安装，所以没有这个数据库。
>
> 　　检查是否安装了 mariadb：[root@localhost ~]# rpm -qa | grep mariadb
>
> 　　如果有就使劲卸载干净：
>
> ​       systemctl stop mariadb
>
> ​       rpm -qa | grep mariadb
>
> ​       rpm -e --nodeps mariadb-5.5.52-1.el7.x86_64
>
> ​       rpm -e --nodeps mariadb-server-5.5.52-1.el7.x86_64
>
> ​       rpm -e --nodeps mariadb-libs-5.5.52-1.el7.x86_64

## 1.3 安装目录信息

/data/mysql						--basedir

/data/mysql/data			  --datadir

# 二、从 mysql 官网下载并上传 mysql安装包

## 2.1 下载 mysql 安装包

小生使用的是 mysql-5.7.26-linux-glibc2.12-x86_64.tar

## 2.2 上传安装文件到 linux 系统

使用 ftp 上传至 linux 系统中

## 2.3 md5校验

安全性校验，和官网提供的MD5值进行校验是有一致

```shell
# md5sum mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz 
08a3b385db2f151598017b63fbcb6c43  mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz
```

# 三、安装 mysql

## 3.1 解压安装包，并移动至 /data 目录下，重命名mysql

解压 mysql 的 gz 安装包：

```shell
# tar -zxvf /data/mysql-5.7.26-linux-glibc2.12-x86_64.tar
```

重命名文件夹

```shell
# mv /data/mysql-5.7.26-linux-glibc2.12-x86_64.tar /data/mysql
```

## 3.2 添加系统用户

添加 mysql 组和 mysql 用户：

```shell
# groupadd mysql     						---添加 mysql 组
# useradd -r -g mysql mysql			---添加 mysql 用户
```

扩展：

　　查看是否存在 mysql 组：`# more /etc/roup | grep mysql`

　　查看 msyql 属于哪个组：`# groups mysql`

　　查看当前活跃的用户列表：`# w`

## 3.3 检查是否安装了 libaio

```shell
# rpm -qa | grep libaio
```

若没有则安装

版本检查：`# yum search libaio`

安装：`# yum -y install libaio`

## 3.4 安装 mysql

进入安装 mysql 软件目录：

```shell
# cd /data/mysql/
```

安装配置文件：

\# vi /etc/my.cnf

如下：

```shell
[mysql] 
# 设置mysql客户端默认字符集 
default-character-set=utf8  
socket=/var/lib/mysql/mysql.sock 
[mysqld] 
#skip-name-resolve 
#设置3306端口 
port = 3306  
socket=/var/lib/mysql/mysql.sock 
# 设置mysql的安装目录 
basedir=/data/mysql 
# 设置mysql数据库的数据的存放目录 
datadir=/data/mysql/data 
# 允许最大连接数 
max_connections=200 
# 服务端使用的字符集默认为8比特编码的latin1字符集 
character-set-server=utf8 
# 创建新表时将使用的默认存储引擎 
default-storage-engine=INNODB 
#lower_case_table_name=1 
max_allowed_packet=16M
```

创建 data 文件夹：

```shell
# mkdir /data/mysql/data
```

修改/data/mysql/data目录拥有者为 mysql 用户：

```shell
# chown -R mysql:mysql /data/mysql/data
```

初始化 mysqld：

```shell
# /data/mysql/bin/mysqld --initialize --user=mysql --basedir=/data/mysql/ --datadir=/data/mysql/data/
```

记住命令行末尾的密码

```shell
# /data/mysql/bin/mysqld --initialize --user=mysql --basedir=/data/mysql/ --datadir=/d ata/mysql/data/ 2019-06-04T21:24:45.813974Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details). 2019-06-04T21:24:46.963280Z 0 [Warning] InnoDB: New log files created, LSN=45790 2019-06-04T21:24:47.314885Z 0 [Warning] InnoDB: Creating foreign key constraint system tables. 2019-06-04T21:24:47.723171Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 2dfb9211-870f-11e9-8e28-4cd98f3f8133. 2019-06-04T21:24:47.759582Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened. 2019-06-04T21:24:47.760176Z 1 [Note] A temporary password is generated for root@localhost: 0PLAA+Bx.1y# 
```

# 四、配置 mysql

## 4.1 设置开机启动

​		a. 复制启动脚本到资源目录：

```shell
# cp /data/mysql/support-files/mysql.server /etc/rc.d/init.d/mysqld
```

​		b. 增加 mysqld 服务控制脚本执行权限：

```shell
# chmod +x /etc/rc.d/init.d/mysqld
```

　　c. 将 mysqld 服务加入到系统服务：

```shell
# chkconfig --add mysqld
```

　　d. 检查mysqld服务是否已经生效：

```shell
# chkconfig --list mysqld
```

命令输出类似下面的结果：

mysqld 0:off 1:off 2:on 3:on 4:on 5:on 6:off 

表明mysqld服务已经生效，在2、3、4、5运行级别随系统启动而自动启动，以后可以使用 service 命令控制 mysql 的启动和停止。

查看启动项：`chkconfig --list | grep -i mysql`

删除启动项：`chkconfig --del mysql`

　　e. 启动 mysqld：

```shell
# service mysqld start
```

> (Starting MySQL.Logging to '/data/mysql/data/localhost.localdomain.err'...**报错看6.2**)

## 4.2 环境变量配置

将mysql的bin目录加入PATH环境变量，编辑 /etc/profile文件：

```shell
# vim /etc/profile
```

profile末尾追加文件如下：

```properties
# mysql env 
export PATH=/data/mysql/bin:$PATH 
```

执行命令使其生效：

```shell
# source /etc/profile
```

用 export 命令查看PATH值：

```shell
# source /etc/profile
```

# 五、登录 mysql

## 5.1 测试登录

登录 mysql：

```shell
# mysql -uroot -p（登录密码为初始化的时候显示的临时密码）
```

初次登录需要设置密码才能进行后续的数据库操作：

```shell
mysql> SET PASSWORD = PASSWORD('123456');（密码设置为了123456）
```

修改密码为 password：

```shell
update user set authentication_string=PASSWORD('password') whereUser='root';
```

## 5.2 防火墙端口偶设置，便于远程访问

**centos7 的firewall防火墙**

```shell
# firewall-cmd --zone=public --add-port=3306/tcp --permanent
# firewall-cmd --reload
```

**centos7 的iptables防火墙**

参考文章尾部

# 六、问题汇总

## 6.1 bin/mysqld: error while loading shared libraries: libnuma.so.1: 安装mysql

如果安装mysql出现了以上的报错信息.这是却少numactl这个时候如果是Centos就yum -y install numactl就可以解决这个问题了. 

ubuntu的就sudo apt-get install numactl就可以解决这个问题了

## 6.2 2019-06-04T21:25:26.462035Z mysqld_safe Directory '/var/lib/mysql' for UNIX socket file don't exists.ERROR! The server quit without updating PID file (/data/mysql/data/localhost.localdomain.pid).

错误原因 该文件目录的所有者 所属组并非我们之前创建的mysql用户

    ```shell
mkdir /var/lib/mysql
chown -R mysql  /var/lib/mysql
    ```

## 6.3 Host is not allowed to connect to this MySQL server解决方法**

先说说这个错误，其实就是我们的MySQL不允许远程登录，所以远程登录失败了，解决方法如下：

1. 在装有MySQL的机器上登录MySQL mysql -u root -p密码
2. 执行use mysql;
3. 执行update user set host = '%' where user = 'root';这一句执行完可能会报错，不用管它。
4. 执行FLUSH PRIVILEGES;

经过上面4步，就可以解决这个问题了。 

> 注: 第四步是刷新MySQL的权限相关表，一定不要忘了，我第一次的时候没有执行第四步，结果一直不成功，最后才找到这个原因。





# Centos6.5 防火墙开放端口

#### 说明

centos6.5处于对安全的考虑，严格控制网络进去。所以在安装mongodb或者使用tomcat，需要开放端口27017或8080。

通常的解决办法有两个。一个是直接关闭防火墙(非常不推荐)：

 ```shell
# ervice iptables stop
 ```

但是这样相当于把系统完全暴露，会带来很大的安全隐患。所以，第二种方案就是修改防火墙策略，如下文介绍。

#### 查看防火墙当前设置

```shell
 # /etc/init.d/iptables status
```

查看已有的防火墙配置信息，需要开放的端口是否已经开放。

配置防火墙策略（root权限）

比如我要开放22/80/3306三个端口，可以在/etc/sysconfig/iptables文件中添加三行信息，如下：

```shell
# vi /etc/sysconfig/iptables
```

内容修改成如下：

```properties
# Generated by iptables-save v1.4.7 on Wed Jun  5 05:36:56 2019
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [7:636]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT

-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 3306 -j ACCEPT

-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
# Completed on Wed Jun  5 05:36:56 2019
```

保存，退出

重启防火墙

 ```shell
# service iptables restart
 ```

完成。可以再执行步骤1查看开放端口配置结果。

#### 开放一个范围的端口

比如开放3000到5000的端口。

```shell
 # -A INPUT -m state –state NEW -m tcp -p tcp –dport 3000:5000 -j ACCEPT 
```

#### 重启或关闭防火墙两种方式

##### 系统重启后生效。开机自启动，或者自动关闭。

开启： `chkconfig iptables on`

关闭： `chkconfig iptables off`

##### 即时生效，系统重启后失效

开启： `service iptables start`

关闭： `service iptables stop`