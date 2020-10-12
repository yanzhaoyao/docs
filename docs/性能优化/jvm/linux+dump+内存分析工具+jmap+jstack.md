# windows+linux下如何使用Memory Analyzer (MAT)进行内存分析（linux+dump+内存分析工具+jmap+jstack）

1.在linux下首先找到tomcat的PID

步骤1：ps aux|grep tomcat_1

步骤2：用jhat生成dump文件，文件后缀为hprof（dump文件后缀的用mat打不开）

jmap -dump:format=b,file=/opt/tomcat6666.hprof 15837

下载到windows下：

sz tomcat6666.hprof

步骤3：下载MAT

http://www.eclipse.org/mat/downloads.php



![img](https://img-blog.csdn.net/20180626184624616?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW5nX3hpYW9tZW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

不需要单独安装eclipse,解压后点击如图exe文件

![img](https://img-blog.csdn.net/20180626184726315?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW5nX3hpYW9tZW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![img](https://img-blog.csdn.net/20180626184815569?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW5nX3hpYW9tZW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



打开刚才的文件tomcat6666.hprof

效果图如下：

![img](https://img-blog.csdn.net/20180626184917143?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW5nX3hpYW9tZW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

备注：

linux下执行 jstack 15837 >zxm.txt

打开zxm.txt，

如下：

![img](https://img-blog.csdn.net/20180626185041689?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW5nX3hpYW9tZW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

————————————————
版权声明：本文为CSDN博主「夜空霓虹」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zhang_xiaomeng/article/details/80819391