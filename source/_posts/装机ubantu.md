title: 装机ubantu
date: 2017-05-09 15:58:00
tags: [ubantu]
category: [装机]
---

## 1. u盘装ubantu12.04

## 2. 左侧栏，打开Ubantu软件中心，搜索chrome，安装之, 如有书签(文件夹)，导入之；
   windows到收藏夹格式转换:iconv -f gb2312 -t utf-8 -o bookmark-utf-8.htm bookmark.htm 
<!--more-->
## 3. 浏览器chrome下载的文件，往往不完整，故，大文件使用firefox来下载，在firefox中添加插件:DownThemAll,然后安装之，
   在网页上右击，即可下载，右击-》dta OneClick打开DownThemAll插件;

## 4. 装一个sougou拼音输入法
   http://pinyin.sogou.com/linux/
   下载: sogoupinyin_1.2.0.0056_amd64.deb
	步骤如下：
	安装指南Ubuntu / Ubuntu Kylin 14.04 LTS 版本只需双击下载的 deb 软件包，即可直接安装搜狗输入法。
	Ubuntu 12.04 LTS 版本由于 Ubuntu 12.04 LTS 自带的 Fcitx 版本较旧，需要先通过 PPA 升级，才能安装下载的 deb 软件包。

	1. 点击左上角的图标打开Dash，输入update-manager，点击更新管理器。

	2. 在更新管理器中，选择菜单：设置->软件源-》其他软件，点击添加...按钮，在弹出的窗口中输入ppa:fcitx-team/nightly， 
	点击添加源。

	3. 然后点击重新载入。

	4. 打开Ubuntu软件中心，在搜索栏输入fcitx，将会搜出fcitx，然后按照一般软件安装步骤安装即可完成升级。


	5. dpkg -i sogoupinyin_1.2.0.0056_amd64.deb完成安装

	6. 注销重新登陆，即可使用搜狗输入法了



## 5. 安装chrome evernote插件;
  下载： http://download.csdn.net/detail/findlyu/8300377
  打开chrome，打开“工具”-》扩展程序，然后将Evernote-Web-Clipper_v6.2.6.crx插件拖进来即可完成安装；

## 6. 安装adt-eclipse
注：装系统或三系统，ubantu12.04 64位可以打开ubantu14.04系统的磁盘，并且可以打开其中的应用程序，也可访问windows磁盘，访问其中的资料;




