title: linux下实用命令
date: 2017-05-09 15:51:35
tags: [工具]
category: [工具]
---

## 1. 查看占用磁盘空间
查看文件夹大小: du -h --max-depth=0 dirname
查看文件大小：du -h filename

## 2. windows到收藏夹格式转换:iconv -f gb2312 -t utf-8 -o bookmark-utf-8.htm bookmark.htm 
<!--more-->
## 3. 去除文件每行的行号（会以空格代替行号）：
sed 's/^[ 0-9]* / /' globalfifo.c > globalfifo1.c
注意一条命令中不能有既作为输入，又作为输出，否则会导致文件为空；如sed 's/^[ 0-9]* / /' globalfifo.c > globalfifo.c，导致文件globalfifo.c为空
去除数字以空字符代替：sed 's/^[ 0-9]* //' globalfifo.c > globalfifo1.c

## 4. chown huyanping:huyanping file; chmod -R 666 dir; cp -r srcdir destdir; 

## 5. 强制清空回收站：sudo rm -fr $HOME/.local/share/Trash/files/*

## 6. 替换代码文件中的中文标点符号，如中文分号；,此时编译会出现:error: stray ‘\357’ in program
```
   sed 's/；/;/' code1.c > code2.c
   替换代码文件中的中文全角空格符号，,此时编译会出现: error: stray ‘\200’ in program和error: stray ‘\343’ in program
   sed 's/　/ /' code1.c > code2.c，注意s后门第一个//之间的字符为中文全角空格，第二个//之间的字符为英文空格
```
	
## 7. 将一个文件所有行以tab键开始
```
sed 's/^[	]*/     /' test4.c > test44.c 
```
其中[]内包含了空格和tab键，最后一个//之间的内容为tab键，linux命令行输入tab键:先按Ctrl+v，再按Tab就可以输入了。
 
## 8. 杀死除ps和bash的第三个进程
```
kill -9 `ps | grep -v bash | grep -v ps | grep -v grep | grep -v awk | awk '{print $1}' | grep -v PID`

ps输出如下:
15088 pts/1    00:00:00 bash
31626 pts/1    00:00:00 a.out
31636 pts/1    00:00:00 ps
```
执行上面命令将会杀死a.out

## 9. 查找某一字符串内容: grep "dm9000" -r * 在当前目录下查找字符串dm9000

## 10. 消除文件中的回车
```
tr -d '\015' < file > newfiel
```
## 11. 定位某个文件: find /home -name "*.txt" -print
用grep搜索一个目录下所有文件中的某个字符串: find /usr -type f -print | xargs grep pid_t


## 12. df -h 查看磁盘使用情况


