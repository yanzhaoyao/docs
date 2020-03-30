# 搭建kafka集群

http://kafka.apache.org/downloads

2.4.0 is the latest release. The current stable version is 2.4.0.

> 准备安装包kafka_2.12-2.4.0.tgz

> 在每台主机上执行下面步骤：

将安装包移到/usr/local目录下

```bash
mv kafka_2.12-2.4.0.tgz /usr/local
```

解压文件

```css
tar -zxvf kafka_2.12-2.4.0.tgz
```

重命名文件夹为kafka

```css
mv kafka_2.11-2.0.0 kafka
```

配置kafka环境变量，首先打开profile文件

```undefined
vim /etc/profile
```

按i进入编辑模式，在文件末尾添加kafka环境变量

```bash
#set kafka environment
export KAFKA_HOME=/usr/local/kafka
PATH=${KAFKA_HOME}/bin:$PATH
```

保存文件后，让该环境变量生效

```bash
bsource /etc/profile
```

> 在kafka-1主机中修改server.properties配置文件

打开配置文件

```bash
vim /usr/local/kafka/config/server.properties
```

修改配置如下（IP地址应该根据实际情况填写）

```cpp
broker.id=1
listeners=PLAINTEXT://192.168.1.42:9092
zookeeper.connect=192.168.1.41:2181,192.168.1.42:2181,192.168.1.47:2181
```

> 在kafka-2主机中修改server.properties配置文件

打开配置文件

```bash
vim /usr/local/kafka/config/server.properties
```

修改配置如下（IP地址应该根据实际情况填写）

```cpp
broker.id=2
listeners=PLAINTEXT://192.168.1.41:9092
zookeeper.connect=192.168.1.41:2181,192.168.1.42:2181,192.168.1.47:2181
```

> 在kafka-3主机中修改server.properties配置文件

打开配置文件

```bash
vim /usr/local/kafka/config/server.properties
```

修改配置如下（IP地址应该根据实际情况填写）

```cpp
broker.id=3
listeners=PLAINTEXT://192.168.1.47:9092
zookeeper.connect=192.168.1.41:2181,192.168.1.42:2181,192.168.1.47:2181
```

> 启动kafka（要确保zookeeper已启动）

在每台主机上分别启动kafka

```bash
/usr/local/kafka/bin/kafka-server-start.sh -daemon config/server.properties 
```

在其中一台虚拟机(192.168.1.47)创建topic

```bash
/usr/local/kafka/bin/kafka-topics.sh --create --zookeeper 192.168.1.47:2181 --replication-factor 3 --partitions 1 --topic test-topic
```

查看创建的topic信息

```bash
/usr/local/kafka/bin/kafka-topics.sh --describe --zookeeper 192.168.1.47:2181 --topic test-topic
```

结果如下图所示：

![topic信息](https://upload-images.jianshu.io/upload_images/12652505-f8c88525c248123f.png)

查看已创建的topic 列表

```shell
/data/kafka_2.12-2.4.0/bin/kafka-topics.sh --list --zookeeper 172.17.64.82:2181
```

![image-20200310170616826](https://tva1.sinaimg.cn/large/00831rSTgy1gcoxzspbk7j31gw0hit9v.jpg)