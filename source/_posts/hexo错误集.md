---
title: hexo错误集
category: "hexo"
date: 2017-04-14 18:29:31
tags: [hexo]
---

## 问题1：出现：ERROR Plugin load failed: hexo-generator-json-content
解决：把node升级到6.0版本及以上，此为还要将.travis.yml中的nodejs版本改为6；
参考文章：[http://www.cnblogs.com/arvin0/p/6664239.html](http://www.cnblogs.com/arvin0/p/6664239.html)

### 解决步骤
>* 1）node-v6.2.0-linux-x64.tar.gz

>* 2）./npm install g hexo
安装完后，node-v6.2.0-linux-x64/bin/node_modules目录即有hexo可执行文件了。

>* 3）将.travis.yum中的nodejs版本改为6

>* 4）博客根目录执行：npm i hexo-generator-json-content --save 

>* 5）发布


