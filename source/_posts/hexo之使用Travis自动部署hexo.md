---
title: hexo之使用Travis自动部署hexo
categories: 工具
date: 2016-10-29 15:29:31
tags: [hexo,travis]
---
[参考文章1](http://blog.csdn.net/xuezhisdc/article/details/53130423)
[参考文章2](https://formulahendry.github.io/2016/12/04/hexo-ci/#)
使用hexo+github搭建完自己的博客之后，每次新增文章，都需要hexo g->hexo d进行发布，此外为了进行备份，也需要将博客目录push到source repo上去(这里就有两个repo：source repo和content repo,我们的github博客网站的真实数据就保存在content repo,而source repo为我们本地博客目录的备份repo),有没有一种解决方案实现push到source repo上去时，自动进行发布呢？答案肯定是yes，这就是hexo的持续集成,实现工具有很多，如：appveyor(针对windows，linux下尝试未成功)，travis等，这里我们使用travis。
<!--more-->

## 目的
>* 一次性实现备份和发布；
>* 方便在不同的电脑上进行博客发布。

## 实现步骤
### 新建Personal Access Tokens
到自己的github上，新建Personal access tokens，选择repo和admin:repo_hook权限，新建文件保存到本地。

### 配置travis ci
实现github账号登陆travis官网，并active自己的source reop，且将“Build only if .travis.yml is present”选项打开。

### 本地安装travis
执行命令：
```
sudo apt-get install ruby2.0
sudo apt-get install rubygems
sudo gem install travis
```
安装时可能出现相关文件版本号大于某个版本的错误，此时执行:sudo gem install xxx。

### 配置travis
git clone下自己的source repo, 到根目录执行:
```
touch .travis.yml
# 这里的 REPO_TOKEN 是变量名,在后面的配置文件中会用到
# TOKEN 是上面github生成的Token
travis encrypt 'REPO_TOKEN=<TOKEN>' --add
# 执行完后，.travis.yml文件中包含如下内容：
env:
  global:
    secure: Kl6AZLqFTGwm3CAktb3Idl2qNstdkC/SdYpyXJcehNnbubfur5Rqpjsy2UplG37+mzZW33UJszBwOrlu/YgNgMKfGPGGs55Wv...
```
加入其他相关脚本到.travis.yml文件中去，完整的.travis.yml内容如下：
```
language: node_js
node_js:
- "4"  # nodejs的版本
branches:
  only:
  - master  # 设置自动化部署的源码分支

env:
  global:
    secure: Kl6AZLqFTGwm3CAktb3Idl2qNstdkC/SdYpyXJcehNnbubfur5Rqpjsy2UplG37+mzZW33UJszBwOrlu/YgNgMKfGPGGs55WvrayaJrjvnXLetdVW4X7WraviCcWFAcHLTtkhF9gmz9DXWYr8FM+rp16AFKU2uWEx1S8/LhxGsRuMwO1SHJqZbWaMUnV88nvg19uN2mfzXzsukmv3cR+rSPa/ybauZj9qBaDb+y+1bsOGECC3TdiyikInRnX3IZDxf+YxSZGqMSl4wXzHWamVZB03LU3AhfUhfZEOhfFpcJG+nnLuS8h092BiEtpGixTewdtpdRPoil3CASGEQ8JvuZ90T/NaiD6m0HTxLZ4zXrWlPLbtCnPXwWli672XcNEiNgZB8COB0IkWv2FQoPl1H7B6UqdwxSxte3aN5YqVfkY7CiJtu+GAFH8u2JSO0OiK+V773vqz+xdt0fB1tXItL2AQtoSrhrr5QchJN5oZhFQpzr/g97O8gBgAQ2p0GLxY3BiFIccGX4hN1Lls/+0WKHPQ3dfFs49jvmelg6fhrYcYQhH/iVJtkIf3TjPq0CvQhiqzMjZoafle9U8RmuY/q/UaoswmkGHQ97qER5QpBWRpDH6JLQOJ63+aiWM53STyrJdraU+vJGGtc5KfTnkRd1Y82c=

before_install:
- export TZ='Asia/Shanghai'  
- npm install -g hexo
- npm install -g hexo-cli 
before_script:
# ------------------------------------------------
# 设置github账户信息 注意修改成自己的信息
# ------------------------------------------------
- git config --global user.name "damotiansheng"
- git config --global user.email 974361900@qq.com
# ------------------------------------------------
# github仓库操作  注意将仓库修改成自己的
# ------------------------------------------------
- sed -i'' "s~https://github.com/damotiansheng/damotiansheng.github.io.git~https://${REPO_TOKEN}:x-oauth-basic@github.com/damotiansheng/damotiansheng.github.io.git~" _config.yml
# 安装依赖组件
install:
- npm install
# 执行的命令
script:
- hexo clean
- hexo generate
# 执行的成功后执行 
after_success:
- hexo deploy
```
### 测试
新建一篇文章，然后执行：
```
git add -A .
git commit -m "xxx"
git push origin master
```
执行文件完后，登陆到[travis官网](https://travis-ci.org)，可以看到类似下面的内容就代表成功了。
```
Worker information
hostname: i-8f64d817-precise-production-2-worker-org-docker.travisci.net:5de14ca3-fcc6-4e64-a6ac-bc0c8a588bd8
version: v2.5.0-8-g19ea9c2 https://github.com/travis-ci/worker/tree/19ea9c20425c78100500c7cc935892b47024922c
instance: 9f2c179:travis:node_js
startup: 481.157573ms
system_info
Build system information
Build language: node_js
Build group: stable
Build dist: precise
Build id: 182467416
Job id: 182467417
travis-build version: 0501eac99
Build image provisioning date and time
Thu Feb  5 15:09:33 UTC 2015
Operating System Details
Distributor ID:	Ubuntu
Description:	Ubuntu 12.04.5 LTS
Release:	12.04
Codename:	precise
Linux Version
3.13.0-29-generic
Cookbooks Version
a68419e https://github.com/travis-ci/travis-cookbooks/tree/a68419e
GCC version
gcc (Ubuntu/Linaro 4.6.3-1ubuntu5) 4.6.3
Copyright (C) 2011 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Apache Maven 3.2.5 (12a6b3acb947671f09b81f49094c53f426d8cea1; 2014-12-14T17:29:23+00:00)
Maven home: /usr/local/maven
Java version: 1.7.0_76, vendor: Oracle Corporation
Java home: /usr/lib/jvm/java-7-oracle/jre
Default locale: en_US, platform encoding: ANSI_X3.4-1968
OS name: "linux", version: "3.13.0-29-generic", arch: "amd64", family: "unix"
fix.CVE-2015-7547
$ export DEBIAN_FRONTEND=noninteractive
nvm install 4
...
hexo clean
INFO  Deleted database.
INFO  Deleted public folder.
hexo deploy
...
Done. Your build exited with 0.
```
此时到自己的github source repo和content repo均可看到最新的提交，访问自己github博客网站也可看到最新的文章了。


