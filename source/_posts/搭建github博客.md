---
title: 搭建github博客
date: 2016-10-14 10:42:33
tags: [搭建github博客]
---
![Alt text](https://damotiansheng.github.io/photos/damotiansheng.png)

### 目的: 利用hexo将个人博客搭建到github上
### [参考的相关文章](http://blog.netpi.me/%E5%AE%9E%E7%94%A8/hexo/)
<!--more-->
### 1、下载node-v4.6.0-linux-x64.tar.xz
该压缩包中包含了编译好的hexo,npm,node可执行文件，可以直接使用

### 2、将hexo可执行文件路径添加到PATH中

### 3、初始化
新建你要存放博客内容的目录，cd到该目录，执行：
``` bash
$hexo init
```
### 4、生成静态页面
``` bash
$hexo generate
```

### 5、本地启动
``` bash
$hexo server
```

### ６、浏览器访问
执行完hexo server之后，浏览器输入http://localhost:4000/查看页面效果

### ７、部署博客到github上去
#### 7.1 新建github仓库，命名规则：你的github账号.github.io
#### 7.2 修改_config.yml
到该文件末尾，添加内容如下：
deploy:
    type: git
    repo: 刚刚创建的github仓库地址.git
    branch: master
注意：type,repo,branch后面有一个空格


#### 7.3 部署到github上
``` bash
$hexo clean
$hexo g
$hexo d
```

#### 7.4 浏览器访问
输入https://你的github账号.github.io即可访问你的博客了。

### 8、hexo命令
#### 8.1 简写
>* hexo n == hexo new
>* hexo g == hexo generate
>* hexo s == hexo server
>* hexo d == hexo deploy

#### 8.2 新建文章
>* 执行hexo new "标题"命令后会在_posts目录会生成文件标题.md，编辑该文件就是编辑该文章

#### 8.3 hexo部署
``` bash
$hexo g
$hexo d
```

### 9、添加评论和头像
#### 9.1 添加评论
>* 到[多说](http://duoshuo.com/create-site/)申请站点
>* 修改theme-yilia下的_config.yml文件，修改为：duoshuo: 站点名称
>* 重新生成并部署即可
>* 如果不行，请参考[Hexo使用多说教程](http://dev.duoshuo.com/threads/541d3b2b40b5abcd2e4df0e9)

#### 9.2 添加头像
>* 在主目录下的source文件夹下(与_posts同目录)新建photos目录，并将xxx.png放到该目录
>* 修改theme-yilia下的_config.yml文件，修改为：avatar: /photos/xxx.png
>* 重新生成并部署即可

### 10、注意
>* 有时hexo new xxx文章后，发布后有乱码，此时删除该文章，然后直接vim新建，然后到博客主目录下执行hexo g -> hexo d，有乱码可能就是由于没有在主目录下执行命令，而在_posts目录下执行命令进行发布了。

