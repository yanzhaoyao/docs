# Mycat宕机了怎么办

基于Mycat分库分表案例demo，来说明Mycat高可用方案

假如mycat所在机器宕机，就算mysql正常，对外提供的服务的mycat挂了之后，整个对外服务就失效了

![1545316007782](http://ww1.sinaimg.cn/large/006tNc79gy1g5xbzxgun6j30tm0g9aeu.jpg)

我们对mycat集群即可

# Mycat之高可用方案Haproxy四层负载

|          | 四层负载均衡                                                 | 七层负载均衡                                                 |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 说明     | 四层负载均衡也称为四层交换机，它主要是通过分析IP层及TCP/UDP层的流量实现的基于IP加端口的负载均衡 | 七层负载均衡器也称为七层交换机，位于OSI（ Open System Interconnection ，开放式系统互联）的最高层，即应用层，此时负载均衡器支持多种应用协议，常见的有HTTP、FTP、SMTP等 |
| 区别     | ![1545316525690](http://ww4.sinaimg.cn/large/006tNc79gy1g5xc0ohohtj305f02vt8h.jpg) | ![1545316549426](http://ww3.sinaimg.cn/large/006tNc79gy1g5xc0883hnj305e02t742.jpg) |
| 效率     | 效率高，直接转发连接                                         | 效率较四层低，先接受连接，然后自己去处理                     |
| 生活案例 | 假如你去公司找李二毛，传达室的老大爷直接告诉你右转直走2号楼自己找去。 | 假如你去公司找李二毛，上市公司前台MM跟你说稍等，然后前台MM自己去找李二毛，此时李二毛跟前台MM说让你等会，前台MM回来后，通知你稍等。类似于这样的传达 |

![1545316893094](http://ww4.sinaimg.cn/large/006tNc79gy1g5xc0h5w9lj30y60ftae9.jpg)



# 用keepalived的VRRP对外提供vip

![1545317095290](http://ww2.sinaimg.cn/large/006tNc79gy1g5xc1fd8btj30x00eadgv.jpg)





# **mycat高可用方案haproxy安装和配置**

## **Haproxy**

1. 下载tar.gz包

   http://www.haproxy.org/download/1.8/src/haproxy-1.8.12.tar.gz （需要科学上网....咳咳咳。）

2. 解压并安装haproxy

​	解压

​	tar -zxvf haproxy-1.8.12.tar.gz

​	安装

​	cd haproxy-1.8.12  

​	将haproxy应用程序安装在/usr/local/haproxy 下

​	make TARGET=linux26 PREFIX=/usr/local/haproxy ARCH=x86_64  

​	make install PREFIX=/usr/local/haproxy

 

3. 配置haproxy

​	创建配置文件

​	vi /usr/local/haproxy/haproxy.cfg 并编辑文件

​	内容如下：

```
global

			log         127.0.0.1 local2

			pidfile     /var/run/haproxy.pid

			maxconn     4000

			daemon

		defaults

			log global

			option dontlognull

			retries 3

			option redispatch

			maxconn 2000

			timeout connect 5000

			timeout client 50000

			timeout server 50000

 

		listen admin_status

			bind 0.0.0.0:1080 

			stats uri /admin ##haproxy自带的管理页面通过http://ip:port/admin访问

			stats auth admin:admin  ##管理页面的用户名和密码

			mode http

			option httplog

 

		listen allmycat_service

			bind 0.0.0.0:8096 ##转发到 mycat 的 8066 端口，即 mycat 的服务端口

			mode tcp

			option tcplog

			option httpchk OPTIONS * HTTP/1.1\r\nHost:\ www

			balance roundrobin

			server mycat_151 192.168.8.151:8066 check port 48700 inter 5s rise 2 fall 3

			server mycat_35 192.168.8.35:8066 check port 48700 inter 5s rise 2 fall 3

			timeout server 20000

 

		listen allmycat_admin

			bind 0.0.0.0:8097 ##转发到 mycat 的 9066 端口，即 mycat 的管理控制台端口

			mode tcp

			option tcplog

			option httpchk OPTIONS * HTTP/1.1\r\nHost:\ www

			balance roundrobin

			server mycat_151 192.168.8.151:9066 check port 48700 inter 5s rise 2 fall 3

			server mycat_35 192.168.8.35:9066 check port 48700 inter 5s rise 2 fall 3

			timeout server 20000
```

4. 配置haproxy的日志输出

​	haproxy采用rsyslog的方式进行日志配置

​	首先安装rsyslog，可通过`rpm -qa|grep rsyslog` 命令判断有没有安装，没有安装自行安装（yum 方式简单明了）

​	`find / -name 'rsyslog.conf'` 找到rsyslog的配置文件

​	`vi rsyslog.conf`

​	将#$ModLoad imudp   

​	  #$UDPServerRun 514  注释解开

​	找到 Save boot messages also to boot.log 在这一行下面加入

​	local2.*  /var/log/haproxy.log

​	保存退出

​	重启rsyslog    `service rsyslog restart`

​	

5. 启动haproxy   

   `/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.cfg`

​	此时出现：

​		proxy allmycat_service has no server available!

​		proxy allmycat_admin has no server available!

​	因为他的check方案没有通过

​		option httpchk OPTIONS * HTTP/1.1\r\nHost:\ www

​		check port 48700 inter 5s rise 2 fall 3

​	上面的配置的意思是，通过http检测的方式进行服务的检测，5s检测一次。

 

## **Xinetd**

通过Xinetd 提供48700端口的http服务来让haproxy进行服务的检测

​	在mycat的服务器上安装xinetd，

安装命令：`yum install xinetd`

​	找到xinetd的配置文件 cat /etc/xinetd.conf 找到 includedir /etc/xinetd.d  

​	进入该目录`cd /etc/xinetd.d`  

​	并新建 mycat_status  shell脚本，`vim mycat_status`

​	内容如下：

```
	service mycat_status   #代表被托管服务的名称

		{

			flags = REUSE

			socket_type = stream				 # socket连接方式

			port = 48700						 # 服务监听的端口

			wait = no							 # 是否并发

			user = root							 # 以什么用户进行启动

			server =/usr/local/bin/mycat_status  # 被托管服务的启动脚本

			log_on_failure += USERID 			 # 设置失败时，UID添加到系统登记表

			disable = no       					 #是否禁用托管服务，no表示开启托管服务

		}
```

​	保存退出

​	

​	创建托管服务启动脚本`/usr/local/bin/mycat_status`

​	`vim /usr/local/bin/mycat_status`

​	内容如下:

​		

```
mycat=`/root/mycat/bin/mycat status |grep 'not running'| wc -l`

		if [ "$mycat" = "0" ];

		then

			/bin/echo -e "HTTP/1.1 200 OK\r\n"

		else

			/bin/echo -e "HTTP/1.1 503 Service Unavailable\r\n"

		fi
```

​	保存退出 并赋予执行权限`chmod +x /usr/local/bin/mycat_status`

​	验证脚本的正确性 `sh /usr/local/bin/mycat_status`  如果返回 `200OK` 字样说明成功

​	

​	加入mycat_status服务

​	`vi /etc/services`

​	在末尾加入以下内容：

​	mycat_status 48700/tcp # mycat_status

​	保存退出，重启xinetd服务，  `service xinetd restart`

​	

​	验证服务是否启动成功

​	`netstat -antup|grep 48700`

​	此时重启haproxy服务即正常

​	------------------------以上为-haproxy 4层代理的负载均衡配置-----------------------

​	----------------------------下面内容为keepalive+haproxy的高可用方案-----------------	

## **keepalive**

​	keepalive安装  （192.168.8.35 -> MASTER   192.168.8.151 -> BACKUP ） 

​	yum install  keepalived

​	find / -name 'keepalived.conf'

​	vi /etc/keepalived/keepalived.conf

​	内容如下

​		! Configuration File for keepalived

​		vrrp_instance VI_1 {

​				state MASTER            #192.168.8.151 上改为 BACKUP

​				interface ens33         #对外提供服务的网络接口

​				virtual_router_id 100   #VRRP 组名，两个节点的设置必须一样，以指明各个节点属于同一 VRRP 组

​				priority 150            #数值愈大，优先级越高,192.168.8.151 上改为比150小的正整数

​				advert_int 1            #同步通知间隔

​				authentication {        #包含验证类型和验证密码。类型主要有 PASS、AH 两种，通常使用的类型为 PASS，据说AH 使用时有问题

​						auth_type PASS

​						auth_pass 1111

​				}

​				virtual_ipaddress { #vip 地址    ens33  通过ifconfig获取

​						192.168.8.233 dev ens33 scope global

​				}

​		} 

​		编辑保存

​	

​	启动keepalived，执行命令 service keepalived start

​	

​	应用通过访问192.168.8.233 即访问抢占了vip（192.168.8.233）的物理机（192.168.8.35）。

​	通过上面的配置，我们可以将前端的请求转到抢占了vip（192.168.8.233）的物理机（192.168.8.35）。但是没有通过监听haproxy的服务，或者说我们没有根据haproxy服务来进行vip的降级。keepalived提供了很多的配置来做服务的检测和降级，但是我们今天不学keepalived的方式我们采用一种定时任务（linux自带的crontab）的方式来做。

​	

​	1.创建check_haproxy.sh 并编辑内容  vi /root/script/check_haproxy.sh

​	内容如下：

​		#!/bin/bash

​		LOGFILE='/root/log/checkHaproxy.log'

​		date >> $LOGFILE

​		count=`ps aux | grep -v grep | grep /usr/local/haproxy/sbin/haproxy | wc -l`

​		if [ $count = 0 ];

​		then

​				echo 'first check fail , restart haproxy !' >> $LOGFILE

​				/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.cfg

​		else

​				exit 0

​		fi

 

​		sleep 3

 

​		count=`ps aux | grep -v grep | grep /usr/local/haproxy/sbin/haproxy | wc -l`

​		if [ $count = 0 ];

​		then

​				echo 'second check fail , stop keepalive service !' >> $LOGFILE

​				service keepalived stop

​		else

​				echo 'second check success , start keepalive service !' >> $LOGFILE

​				keepalived=` ps aux | grep -v grep | grep /usr/sbin/keepalived | wc -l`

​				if [ $count = 0 ];

​				then

​					service keepalived start

​				fi

​				exit 0

​		fi

​	

​	2.执行 crontab -e 编辑定时任务 每一分钟检测haproxy服务存活，如果服务启动不了，停掉keepalived服务， vip即转发至backup的192.168.8.151的机器

​	* * * * * sh /root/script/check_haproxy.sh	

​	