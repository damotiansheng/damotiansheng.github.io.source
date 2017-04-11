---
title: nginx总结
date: 2017-04-11 15:29:31
tags: [nginx]
---

## 前言
通过阅读网上nginx相关博客以及下载的nginx源码，对nginx有了一定的了解，现进行一些总结，总结如下所示。<!--more-->

## 调试
1）gdb -p pid进行调试pid进程；
2）gdb不停收到sigtrap信号：
设置断点之后，有时总会收到上面的信号，导致无法执行到断点位置，解决方法，在收到该信号时，执行：
set $ps&=~(1<<8)即可。

## 使用内存池思想
新来连接直接从连接结构体内存池取一个结构体用，释放则挂上去即可，无须每次来连接就申请内存，且这里为大块内存，避免了内存碎片，连接结构体里面有事件结构体，监听读写也只需加入该结构体中的信息到epoll即可，无需每次监听某个读写事件都申请内存。如：Ngxin对于每个建立成功的TCP连接会预先分配一个内存池。也就是ngx_connection_t结构体中的pool，在处理HTTP请求时，将会为每个请求分配一个内存池。

## 使用位域节约空间。

## 使用accept锁避免惊群问题。

## 使用红黑树处理超时事件。

## master进程管理多个worker进程，多个worker进程处理连接，并发量加大。

## 使用自己缓存的时间，避免gettimeofday（需要发送中断给linux内核，内核需要做进程间切换来处理这个调用）系统调用的频繁调用。

## 采用事件驱动模型，依据不同的运行平台，选择合适的IO复用函数，如epoll。

## 具有自动重启worker进程的功能，即发现某个worker进程退出了，自动开启一个新的worker进程。

## 具有不中断服务的情况下平滑升级nginx程序的功能。

## 具有不中断服务器的情况下重新加载配置文件，使配置文件马上生效的功能。

## 采用模块化思想，可以添加/删除外部模块，具有更大的扩展性。

## 采用socketpair创建channel[2]用于master与worker以及worker之间（只不过这里没有通信）的通信。

## 可以托管网站以及作为反向代理。

## nginx使用非阻塞模式，将一个HTTP请求分成多个阶段，细化处理，并不会阻塞在某一个读/写上，而是阻塞在epoll上以提高并发。

## 创建一个监听fd（调用socket，bing，listen）之后，只要某个进程能得到该fd，该进程就可以accept它得到连接。


## nginx通过尽量减少内存拷贝来提高效率
在整个解析http请求的状态机中始终遵循着两条重要的原则：减少内存拷贝和回溯。内存拷贝是一个相对比较昂贵的操作，大量的内存拷贝会带来较低的运行时效率。nginx在需要做内存拷贝的地方尽量只拷贝内存的起始和结束地址而不是内存本身，这样做的话仅仅只需要两个赋值操作而已，大大降低了开销，当然这样带来的影响是后续的操作不能修改内存本身，如果修改的话，会影响到所有引用到该内存区间的地方，所以必须很小心的管理，必要的时候需要拷贝一份。这里不得不提到nginx中最能体现这一思想的数据结构，ngx_buf_t，它用来表示nginx中的缓存，在很多情况下，只需要将一块内存的起始地址和结束地址分别保存在它的pos和last成员中，再将它的memory标志置1，即可表示一块不能修改的内存区间，在另外的需要一块能够修改的缓存的情形中，则必须分配一块所需大小的内存并保存其起始地址，再将ngx_bug_t的temprary标志置1，表示这是一块能够被修改的内存区域。

## 一个nginx进程可以并发的处理处于不同阶段的多个请求。

## nginx使用到的进程间通信和同步方法
进程间通信：套接字、共享内存、信号；
同步方法：原子操作、信号量、文件锁；

## nginx中的负载均衡算法
实现代码很不错，如加权轮询算法和IP哈希算法，详情见博客：[http://blog.csdn.net/column/details/nginxroad.html](http://blog.csdn.net/column/details/nginxroad.html)
注：这里的负载均衡算法指的是如何从一些后台服务器和备用服务器中选择一台服务器作为处理请求的服务器。

## 分配内存时会进行内存对齐，如ngx_pmemalign函数。

## 优秀的数据结构
红黑树ngx_rbtree_t、基数树ngx_radix_tree_t（字典树）、动态数组ngx_array_t、单向链表ngx_list_t、双向链表ngx_queue_t、哈希表ngx_hash_t、缓冲区链表ngx_chain_t、内存池ngx_pool_t等，各种实现值得参考。


## nginx针对http请求数据无法一次性读取完时如何处理该http请求？
ngx_event_t *rev，rev->data保存ngx_connection_t* c指针，ngx_http_wait_request_handler函数中会将c->data设置为ngx_http_request_t
指针，代表当前处理的请求，这样当用户来请求了，而数据一次性无法读取完，设置对应阶段的回调函数，当再次有数据到来时，nginx会通过rev得到先前的请求ngx_http_request_t，从而得到先前已经读取到的数据，这些数据是保存在ngx_http_request_t结构体中的。

## nginx系列博客：
[http://blog.csdn.net/yusiguyuan/article/category/2091295](http://blog.csdn.net/yusiguyuan/article/category/2091295)
[http://blog.csdn.net/column/details/nginxroad.html](http://blog.csdn.net/column/details/nginxroad.html)
[http://blog.csdn.net/jy02326166/article/category/2332871](http://blog.csdn.net/jy02326166/article/category/2332871)
[http://blog.csdn.net/livelylittlefish](http://blog.csdn.net/livelylittlefish)
[http://blog.csdn.net/chen19870707/article/category/2647301](http://blog.csdn.net/chen19870707/article/category/2647301)



