---
title: fastdfs连接池
date: 2016-12-03 17:51:25
tags: [fastdfs,连接池]
---

# fastdfs连接池

------
## 思想
为了提高连接速度，fastdfs使用到了连接池，并用hash数组来加速查找，key为ip_port, value为：ConnectionManager（见后面），当需要进行连接某个ip:port时，以该ip_port作为key，到hash数组中查找，得到value:ConnectionManager,该结构中保存了针对该ip:port的已建立连接的socket，第一次连接时，会调用conn_pool_connect_server函数建立连接，当需要关闭某个连接时，会调用conn_pool_close_connection_ex函数来将该连接放入连接池中或者直接关闭该连接，放入连接池中即会插入到hash数组中去，此外还会依据该连接的上次访问时间，如果很久没有访问该连接了，就会关闭该连接，见conn_pool_get_connection函数，通过hash数组+关闭连接时将该连接保存到hash数组中来实现连接池，提高连接速度；
<!--more-->
## 用到的数据结构
```
typedef struct
{
	int sock;
	int port;
	char ip_addr[INET6_ADDRSTRLEN];
    int socket_domain;  //socket domain, AF_INET, AF_INET6 or PF_UNSPEC for auto dedect
} ConnectionInfo;

struct tagConnectionManager;

typedef struct tagConnectionNode {
	ConnectionInfo *conn;  // pointer the end of ConnectionNode, see conn_pool_get_connection func
	struct tagConnectionManager *manager;
	struct tagConnectionNode *next; // pointer to next ConnectionNode which is the same ip and port in conn
	time_t atime;  //last access time, if 
} ConnectionNode;

// connection pool, key is ip_port, value is ConnectionManager, see conn_pool_get_connection func
typedef struct tagConnectionManager {
	ConnectionNode *head;  // head pointer to the first ConnectionNode
	int total_count;  //total connections，已经建立的连接数，超过一定数值时，不会再建立连接
	int free_count;   //free connections
	pthread_mutex_t lock;
} ConnectionManager;

typedef struct tagConnectionPool {
	HashArray hash_array;  //key is ip_port, value is ConnectionManager, see conn_pool_get_key func
	pthread_mutex_t lock;
	int connect_timeout;
	int max_count_per_entry;  //0 means no limit

	/*
	connections whose the idle time exceeds this time will be closed
    unit: second
	*/
	int max_idle_time;
    int socket_domain;  //socket domain
} ConnectionPool;
```

## 相关代码
connection_pool.h/connection_pool.c
核心函数：conn_pool_get_connection



