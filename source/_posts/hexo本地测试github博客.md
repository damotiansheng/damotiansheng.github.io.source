---
title: hexo本地测试github博客
date: 2017-04-19 16:51:54
tags: [hexo]
category: "hexo"
---

## 执行如下命令

```
git clone https://github.com/damotiansheng/damotiansheng.github.io.source.git
cd damotiansheng.github.io.source/
cp _config.yml _config.yml.bak
npm install hexo --save
npm install hexo-server --save
hexo init .  //会覆盖_config.yml
cp _config.yml.bak _config.yml
hexo clean
hexo g
hexo server
```

## 验证
访问[http://localhost:4000/](http://localhost:4000/)查看效果即可


