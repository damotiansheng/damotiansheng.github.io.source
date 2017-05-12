---
title: c与c++相互调用
date: 2017-05-12 13:58:26
tags: [c,c++]
category: [c/c++]
---


# 目的 
实现C++中调用C函数以及C调用C++函数。

## C++中调用C函数的实例
```
// cHeader.h
#ifndef C_HEADER
#define C_HEADER
void print( int i );
#endif
```
<!--more-->
```
//cHeader.c
#include <stdio.h>
#include "cHeader.h"

void print( int i )
{
	printf( "cHeader %d\n", i );
}
```

```
//C++.cpp
extern "C"
{
	#include "cHeader.h"
}

int main()
{
	print(3);
	return 0;
}
```

编译执行：
```
gcc -c cHeader.c
g++ C++.cpp cHeader.o
./a.out
cHeader 3
```

## C中调用C++函数的实例
```
// cppHeader.h
#ifndef CPP_HEADER
#define CPP_HEADER

extern "C" 
{
	void print( int i );
}

#endif
```

```
// cppHeader.cpp
#include "cppHeader.h"

//#include <iostream>  //不能有c++专有的头文件，否则无法编译
#include <stdio.h>
using namespace std;


void print( int i )
{
//	cout << "cppHeader " << i << endl; //不能有c++专有的函数，否则无法编译
	printf( "%d\n", i );
}
```

```
// c.c
extern void print( int i );

int main()
{
	print(3);
	return 0;
}
```
编译执行：
```
g++ -c cppHeader.cpp
gcc c.c cppHeader.o  //不能使用g++ c.c cppHeader.o，否则会出现undefined reference to `print(int)'
./a.out
3
```
若C调用的C++函数中包含了cout等C++符号，则需要做一个中间接口库，对C++库进行二次封装，实例如下：
```
// world.cpp

#include <iostream>

void world( int )
{
  std::cout << "world" << std::endl;  
}

/*
使用nm命令查看导出的函数，发现world被修饰后的完整名称为T _Z5worldi, C++默认会返回一个整型，所以前面有T
nm -D libworld.so 
0000000000201050 B __bss_start
                 U __cxa_atexit
                 w __cxa_finalize
0000000000201050 D _edata
0000000000201058 B _end
00000000000009c8 T _fini
                 w __gmon_start__
00000000000007c0 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 w _Jv_RegisterClasses
0000000000000935 T _Z5worldi
                 U _ZNSolsEPFRSoS_E
                 U _ZNSt8ios_base4InitC1Ev
                 U _ZNSt8ios_base4InitD1Ev
                 U _ZSt4cout
                 U _ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
                 U _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
*/

```

```
// mid.cpp

#include <iostream>

void world(int); // 会以C++风格调用world函数

#ifdef __cplusplus
extern "C" {  // 即使这是一个C++程序，下列这个函数的实现也要以C约定的风格来搞！
#endif

  void m_world() // 会生成C风格函数名
  {
    world(1);
  }

#ifdef __cplusplus
}
#endif
/*
使用nm命令查看导出的函数，发现world被修饰后的完整名称为U _Z5worldi
nm -D libmid.so 
0000000000201048 B __bss_start
                 U __cxa_atexit
                 w __cxa_finalize
0000000000201048 D _edata
0000000000201050 B _end
0000000000000884 T _fini
                 w __gmon_start__
00000000000006b0 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 w _Jv_RegisterClasses
0000000000000815 T m_world
                 U _Z5worldi
                 U _ZNSt8ios_base4InitC1Ev
                 U _ZNSt8ios_base4InitD1Ev
*/

```

```
// test.c

#include <stdio.h>

int main()
{
  m_world();

  return 0;
}
```

执行命令编译执行：
```
g++ --shared -fPIC -o libworld.so world.cpp
sudo cp libworld.so /lib/
g++ --shared -fPIC -o libmid.so mid.cpp -lworld
sudo cp libmid.so /lib/
gcc test.c -l mid -o test
./test
world
```

## 参考文章
[http://www.cnblogs.com/this-543273659/archive/2011/08/19/2146022.html](http://www.cnblogs.com/this-543273659/archive/2011/08/19/2146022.html)
[https://wenku.baidu.com/view/1a851168aef8941ea66e0510.html](https://wenku.baidu.com/view/1a851168aef8941ea66e0510.html)
[http://www.cppblog.com/wolf/articles/77828.html](http://www.cppblog.com/wolf/articles/77828.html)
[http://www.jianshu.com/p/8d3eb96e142a#](http://www.jianshu.com/p/8d3eb96e142a#)




