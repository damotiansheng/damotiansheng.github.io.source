title: hadoop之debug
date: 2017-05-09 15:45:08
tags: [hadoop]
category: [hadoop]
---

## 1、hadoop dfsadmin -safemode leave离开安全模式后，eclipse之就可以删除文件了；

## 2、hadoop文件夹下的文件权限不能随便改变，如sudo chmod -R a+rw hadoop/ 可能会导致datanode启动不了；
<!--more-->
从而导致：hadoop dfsadmin -report，输出:
```
Configured Capacity: 0 (0 KB)
Present Capacity: 0 (0 KB)
DFS Remaining: 0 (0 KB)
DFS Used: 0 (0 KB)
DFS Used%: �%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
```
总之，那里出问题，看logs文件夹下对应的日志文件


## 3、Hadoop运行class类出现Exception in thread "main" java.lang.NoClassDefFoundError
这个问题一直让我自己写的class类无法再hadoop平台运行，困惑好几天了，看权威指南的进度直接无进展。
解决方法是在 conf/hadoop-env.sh添加hadoop的类路径
export HADOOP_CLASSPATH=.
还有用javac编译hadoop平台的java代码要用到好多的hadoop的class类，一直写 javac -classpath ...
很麻烦，可以在 ~/.bashrc 中添加如下语句：
export CLASSPATH="$CLASSPATH:$HADOOP_HOME/hadoop-core-0.20.203.0.jar:$HADOOP_HOME/lib/commons-cli-1.2.jar"

from: [http://blog.chinaunix.net/uid-27672830-id-3374186.html](http://blog.chinaunix.net/uid-27672830-id-3374186.html)

