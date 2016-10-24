---
title: fastdfs文件系统搭建
date: 2016-10-21 16:54:20
tags: [fastdfs,分布式文件系统]
---

## ubantu14.04 x64下搭建FastDFS分布式存储环境（使用Nginx模块）
<!--more-->
### 1. 软件准备
#### 1.1 从[happyfish100](https://github.com/happyfish100)下载最新的fastdfs（当前为v5.08）、libfastcommon、fastdfs-nginx-module；
#### 1.2 从nginx官网下载nginx,我下载的版本为nginx-1.10.2.tar.gz

### 2. 安装libfastcommon
#### 2.1 安装
```bash
#./makesh
#./make.sh install
```
#### 2.2 配置
但是FastDFS主程序设置的lib目录是/usr/local/lib
所以需要创建软链接.
```bash
ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so 
```
### 3. 安装fastdfs主程序
#### 3.1  安装
进入fastdfs主目录，依次执行：
```bash
./make.sh 
./make.sh install
```

执行完毕后，可执行文件在/usr/bin/下以fdfs开头，ll /usr/bin/fdfs*，会看到一些可执行程序，如fdfs_upload_file等；
在/etc/fdfs/目录下也有一些配置文件，如：storage.conf.sample等；
执行： cp fastdfs/conf/* /etc/fdfs，即将conf目录下的所有文件复制到/etc/fdfs目录下

#### 3.2 配置
在fastdfs同级目录，新建目录storage_base_path，client_base_path，mod_fastdfs_base_path，store_path0，tracker_base_path
修改/etc/fdfs/tracker.conf(若没有，则cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf)
将base_path的值设置为tracker_base_path目录，如：
```bash
vim /etc/fdfs/tracker.conf
base_path=/home/hyp/Desktop/opensource/tracker_base_path
http.server_port=8080

vim /etc/fdfs/storage.conf
group_name=group1
base_path=/home/hyp/Desktop/opensource/storage_base_path
store_path0=/home/hyp/Desktop/opensource/storage_base_path
tracker_server=172.16.55.156:22122
http.server_port=8080


vim /etc/fdfs/client.conf
base_path=/home/hyp/Desktop/opensource/client_base_path
tracker_server=172.16.55.156:22122
```

#### 3.3 测试
```bash
which fdfs_storaged 
/usr/bin/fdfs_storaged

fdfs_trackerd /etc/fdfs/tracker.conf restart
netstat -antp | grep trackerd

fdfs_storaged /etc/fdfs/storage.conf restart
netstat -antp | grep storage

ps -ef|grep fdfs

fdfs_upload_file client.conf ~/Desktop/fork.c
group1/M00/00/00/rBA3nFgJeMCAUOjoAAAAxxbGNXU56061.c
fdfs_upload_file client.conf ~/Desktop/kenan1.jpg 
group1/M00/00/00/rBA3nFgJxfKABNobAAITVP28J6s210.jpg

终止命令：
killall fdfs_storaged 
killall fdfs_trackerd 
```

### 4. 安装fastdfs-nginx-module
```bash
修改fastdfs-nginx-module的config文件
原来的内容是
CORE_INCS="$CORE_INCS /usr/local/include/fastdfs /usr/local/include/fastcommon/"

vim /home/nginx/fastdfs-nginx-module/src/config,修改为
CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon"

注：貌似github上的fastdfs-nginx-module无需修改
```

### 5. 安装nginx并运行
#### 5.1 解压nginx-1.10.2，并执行：
```bash
./configure --prefix=/usr/local/nginx --add-module=/home/hyp/Desktop/opensource/fastdfs-nginx-module/src
make
make install
```
#### 5.2 配置
```bash
cp fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs/

vim /usr/local/nginx/conf/nginx.conf
将server字段的listen字段修改为8080
并添加：
location /group1/M00 {
            root /home/hyp/Desktop/opensource/storage_base_path/data;
            ngx_fastdfs_module;
        }

注：/home/hyp/Desktop/opensource/storage_base_path/data该目录是指向真正存储文件的地方

修改fastdfs的nginx模块的配置文件mod_fastdfs.conf:
vim /etc/fdfs/mod_fastdfs.conf
base_path=/home/hyp/Desktop/opensource/mod_fastdfs_base_path      #保存日志目录
tracker_server=172.16.55.156:22122    #tracker 服务器的 IP 地址以及端口号
storage_server_port=23000            #storage 服务器的端口号
group_name=group1                    #当前服务器的 group 名
url_have_group_name = true           #文件 url 中是否有 group 名
store_path_count=1                   #存储路径个数，需要和 store_path 个数匹配
store_path0=/home/hyp/Desktop/opensource/mod_fastdfs_base_path    #存储路径
http.need_find_content_type=true     # 从文件 扩展 名查 找 文件 类型 （ nginx 时 为true）,好像没有该字段
group_count = 1                      #设置组的个数

然后在末尾添加分组信息，目前只有一个分组，就只写一个
[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/home/hyp/Desktop/opensource/storage_base_path

建立M00至存储目录的符号连接
ln -s /home/hyp/Desktop/opensource/storage_base_path/data /home/hyp/Desktop/opensource/storage_base_path/M00 （好像删除也可以，因为上面
/group1/M00的root已经指向data了）
```

#### 5.3 运行
```bash
fdfs_trackerd /etc/fdfs/tracker.conf restart
fdfs_storaged /etc/fdfs/storage.conf restart
fdfs_upload_file /etc/fdfs/client.conf ~/Desktop/kenan.jpg 
group1/M00/00/00/rBA3nFgJuvmAHHJ0AAENmFEdiOY703.jpg
到/usr/local/nginx/sbin目录，执行: ./nginx
./nginx
ngx_http_fastdfs_set pid=22882

lsof -i:8080
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   22883   root    6u  IPv4 860115      0t0  TCP *:http-alt (LISTEN)
nginx   22884 nobody    6u  IPv4 860115      0t0  TCP *:http-alt (LISTEN)

输入http://localhost:8080/即可显示首页
输入http://localhost:8080/group1/M00/00/00/rBA3nFgJuvmAHHJ0AAENmFEdiOY703.jpg即可显示kenan.jpg图片了
```

### 6. 遇到的问题
#### 6.1 官网下载最新版本后，我下载的为nginx-1.10.2.tar.gz，编译运行
```bash
ps -aux|grep nginx
出现：
root      7421  0.0  0.0  28588   564 ?        Ss   09:42   0:00 nginx: master process ./nginx
nobody    7422  0.0  0.0  29012  2552 ?        S    09:42   0:00 nginx: worker process
```
但是在我ubantu12.04 x64的笔记本上只出现了一个进程，怪了，在ubantu12.04上，添加模块--add-module=/home/hyp/Desktop/opensource/fastdfs-nginx-module/src
后，也只有一个进程了，且此时浏览器访问[localhost](http://localhost)时，总是无法显示，浏览器总在转圈，可能原因是配置文件中的http.server_port端口没有配
置好，端口需要配置与nginx中监听的端口一致，或者是mod_fastdfs.conf文件没有配置好，如末尾没有添加分组信息，此问题困扰了两天；

#### 6.2 有时执行: fdfs_storaged storage.conf
```bash
[2016-10-21 10:30:12] ERROR - file: shared_func.c, line: 968, /storage.conf is not a regular file
root@hyp-HP-Pro-3340-MT:/etc/fdfs# [2016-10-21 10:30:12] ERROR - file: storage_func.c, line: 1076, load conf file "storage.conf" fail, ret code: 22
[2016-10-21 10:30:12] CRIT - exit abnormally!

^C
[2016-10-21 10:03:10] ERROR - file: shared_func.c, line: 968, /./storage.conf is not a regular file
[2016-10-21 10:03:10] ERROR - file: storage_func.c, line: 1076, load conf file "./storage.conf" fail, ret code: 22
[2016-10-21 10:03:10] CRIT - exit abnormally!

^C

使用完整路径即可： fdfs_storaged /etc/fdfs/storage.conf
```

### 7. 参考来源：

[http://www.tuicool.com/articles/q6ZvUn](http://www.tuicool.com/articles/q6ZvUn)

[http://www.tuicool.com/articles/q6ZvUn](http://blog.itpub.net/7734666/viewspace-1292485/)






