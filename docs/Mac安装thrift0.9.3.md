# Mac安装指定版本thrift-0.9.3

通过brew安装的thrift都是最新版，而团队里使用的版本是0.9.3；因担心通过RPC访问产生兼容性问题，自己踩了好多坑顺便记录一下---怎么在mac下安装降级版的thrift。

# 一、查询Thrift依赖包

1） 查看thrift所依赖的安装包

`brew info thrift`

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1geq327lx1zj30ge085wfr.jpg)

2）通过brew 安装依赖包

`brew install boost openssl libevent`

3）安装bison 2.5版本以上

通过brew安装bison的时候(thrift-0.9.3版本及以上)，会报“configure: error: Bison version 2.5 or higher must be installed on the system!”，故需要直接下载tar包安装

bison官网地址：http://www.gnu.org/software/bison/

笔者下载最新的：http://ftp.gnu.org/gnu/bison/bison-3.5.tar.gz

`wget http://ftp.gnu.org/gnu/bison/bison-3.5.tar.gz`

`tar -zxvf bison-3.5.tar.gz`

`cd bison-3.5`

`./configure`

`make && make install`

**注意：这步结束后需要查询当前版本**

**bison --version**

**如版本不对或者报错Abort trap: 6，则说明安装有问题**

**在网上查了很多内容，有些办法是替换xcode自带的bison，但是这种办法并不适用；**

**笔者的解决办法是指定刚刚安装的目录**

**1，先通过brew list bison 查询当前安装的版本和路径**

**2，使用which bison查询当前执行的程序目录**

**3，将brew list bison 查出的路径写入/etc/profile，覆盖which bison错误的路径**

**export PATH=/usr/local/Cellar/bison/3.3.2/bin:$PATH**

# 二、安装thrift

thrift官网链接：http://archive.apache.org/dist/thrift/

`wget http://archive.apache.org/dist/thrift/0.9.3/thrift-0.9.3.tar.gz `

`tar -zxvf thrift-0.9.3.tar.gz`

`cd thrift-0.9.3`

`./configure`

`make`

`make install `

# 三、验证

```shell
$ thrift -version
Thrift version 0.9.3
```

安装成功