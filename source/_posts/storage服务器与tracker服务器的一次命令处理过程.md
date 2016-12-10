---
title: storage服务器与tracker服务器的一次命令处理过程
date: 2016-12-10 16:52:25
tags: [fastdfs]
---
本篇文章简单介绍下storage server(client端)向tracker server(server端)发送一个命令，然后server端处理命令，client端得到结果的流程。
<!--more-->

## 具体过程
storage server通过向server端建立连接，发送一个带包头的数据包，tracker server主线程监听得到该fd，并将该fd按轮询写入到某一个管道，从而选择一个work线程来处理，该work线程调用recv_notify_read函数得到fd，接着设置READ事件并使用IO复用函数（如epoll）进行监听，READ事件的处理函数为client_sock_read，当client端发送命令过来时，该函数被调用，当接收完一个完整的命令包时会调用tracker_deal_task(pTask)函数进行命令的分发处理，调用client_sock_write函数发送response, 发送完毕后，继续监听该fd的读事件，而storage server则会读取得到reponse数据，从而得到命令的结果。

## 举例
[来源于](http://blog.csdn.net/lctel/article/details/12752803)
storage server发送TRACKER_PROTO_CMD_STORAGE_GET_SERVER_ID命令：
storage进程main->storage_func_init->storage_func_init->tracker_get_my_server_id->tracker_get_storage_id->tcpsenddata_nb(TRACKER_PROTO_CMD_STORAGE_GET_SERVER_ID)/fdfs_recv_response
tracker作服务端处理：
tracker_deal_task(TRACKER_PROTO_CMD_STORAGE_GET_SERVER_ID)->tracker_deal_get_storage_id->fdfs_get_storage_id_by_ip
