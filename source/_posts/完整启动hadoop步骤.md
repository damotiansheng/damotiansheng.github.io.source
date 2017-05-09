title: 完整启动hadoop步骤
date: 2017-05-09 15:33:16
tags: [hadoop]
category: [hadoop]
---

# 目的
有时启动hadoop会出现各种问题，现给出完整的启动hadoop步骤，搭建hadoop,请参考：[http://blog.csdn.net/zhaoyl03/article/details/8657104](http://blog.csdn.net/zhaoyl03/article/details/8657104)。
<!--more-->

## 启动步骤如下
>* 1）jps查看namenode等是否已经启动，若是，则kill之；

>* 2）切换用户: su - hadoop(因为开始我执行过: su chown -R hadoop:hadoop hadoop，即将hadoop目录属主和用户组全部改为hadoop用户和hadoop用户组了,为了防止权限问题，切换为hadoop用户);

>* 3）清空hadoop.tmp.dir目录和/tmp/hadoop*： 
查看conf/core-site.xml文件，查看hadoop.tmp.dir值，我的为/usr/local/hadoop/tmp
sudo rm -rf /tmp/hadoop*
sudo rm -rf /usr/local/hadoop/tmp/*

>* 4）ps -e | grep namenode等，ps -e | grep java，kill掉java进程等相关进程，否则启动时可能会出现:
 ERROR org.apache.hadoop.hdfs.server.namenode.NameNode: java.io.IOException
: Cannot lock storage /usr/local/hadoop/datalog1. The directory is already locked.

>* 5）格式化: bin/hadoop namenode -format
出现下面相关内容表示成功：
15/05/28 15:52:25 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at hulin-HP-Pro-3380-MT/127.0.1.1
************************************************************/


>* 6）启动: bin/start-all.sh

>* 7）jps查看java进程，启动成功如下：
27950 NameNode
28811 TaskTracker
28896 Jps
28559 JobTracker
28469 SecondaryNameNode
28199 DataNode

>* 8）若某一进程启动失败，查看logs文件夹下的.log文件，找出异常，百度谷歌之即可解决；

