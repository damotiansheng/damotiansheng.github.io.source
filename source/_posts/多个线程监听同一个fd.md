---
title: 多个线程监听同一个fd
date: 2016-11-27 16:23:59
tags: [multi pthread]
---

## 多个线程监听同一个fd

### 测试环境
ubantu 12.04 x64

### 目的
多个线程监听同一fd,看看是否会出现问题，连接到来时，是否每个线程都会被唤醒，
还是只唤醒一个，每个线程被唤醒的次数是否会相等。
<!--more-->
### 代码
```bash
// server.c
#include <sys/time.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 4321
#define BACKLOG 100
#define MAXRECVLEN 1024
pthread_t tid[10];
long cnt[10];

void accept_thread_entrance( void* arg );

int main(int argc, char *argv[])
{
    char buf[MAXRECVLEN];
    int listenfd, connectfd;  /* socket descriptors */
    struct sockaddr_in server; /* server's address information */
    struct sockaddr_in client; /* client's address information */
    socklen_t addrlen;
    /* Create TCP socket */
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
    {
        /* handle exception */
        perror("socket() error. Failed to initiate a socket");
        exit(1);
    }
 
    int opt = SO_REUSEADDR;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    server.sin_addr.s_addr = htonl(INADDR_ANY);
    if(bind(listenfd, (struct sockaddr *)&server, sizeof(server)) == -1)
    {
        perror("Bind() error.");
        exit(1);
    }
    
    if(listen(listenfd, BACKLOG) == -1)
    {
        perror("listen() error. \n");
        exit(1);
    }
        int i = 0;
	int result;

    for (i=0; i<5; i++)
    {
	cnt[i] = 0;
	
				if ((result=pthread_create(&tid[i], NULL, \
					accept_thread, \
					(void *)(long)listenfd)) != 0)
				{
                                        puts("pthread_create error");
					break;
				}
    }

  
    printf( "%lu, %lu, %lu, %lu, %lu \n", tid[0], tid[1], tid[2], tid[3], tid[4] ); //这里的输出与子线程的pthread_self返回值并不一样

    while(1);

    return 0;
}

pthread_mutex_t mutex;
pthread_t childTid[10];
int idx = 0;

void accept_thread( void* arg )
{
	int listenfd = (long)arg;
	int connectfd;
	struct sockaddr_in client;
        socklen_t addrlen;
	addrlen = sizeof(client);
    pthread_mutex_lock(&mutex);
    childTid[idx++] = pthread_self();
    pthread_mutex_unlock(&mutex);

    while(1)
    {
        if((connectfd=accept(listenfd,(struct sockaddr *)&client, &addrlen))==-1)
          {
            perror("accept() error. \n");
            exit(1);
          }

        struct timeval tv;
        gettimeofday(&tv, NULL);
        printf("pthread-id=%lu,You got a connection from client's ip %s, port %d at time %ld.%ld\n",pthread_self(), inet_ntoa(client.sin_addr),htons(client.sin_port), tv.tv_sec,tv.tv_usec);
	int i = 0;
  	pthread_mutex_lock(&mutex);
        for( i = 0; i < 5; i++ )
	{
		if( pthread_self() == childTid[i] )
		{
			cnt[i]++;
			break;
		}
	}

	printf( "%ld, %ld, %ld, %ld, %ld \n", cnt[0], cnt[1], cnt[2], cnt[3], cnt[4] );
        pthread_mutex_unlock(&mutex);
        close(connectfd);
    }
	return;
}
```

```bash
// client.c:
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>  /* netdb is necessary for struct hostent */
#define PORT 4321  /* server port */
#define MAXDATASIZE 100

int main(int argc, char *argv[])
{
    int sockfd, num;    /* files descriptors */
    char buf[MAXDATASIZE];    /* buf will store received text */
    struct hostent *he;    /* structure that will get information about remote host */
    struct sockaddr_in server;
    if (argc != 2)
    {
        printf("Usage: %s <IP Address>\n",argv[0]);
        exit(1);
    }
    
    if((he=gethostbyname(argv[1]))==NULL)
    {
        printf("gethostbyname() error\n");
        exit(1);
    }
    
    if((sockfd=socket(AF_INET,SOCK_STREAM, 0))==-1)
    {
        printf("socket() error\n");
        exit(1);
    }
    bzero(&server,sizeof(server));
    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    server.sin_addr = *((struct in_addr *)he->h_addr);
    if(connect(sockfd, (struct sockaddr *)&server, sizeof(server))==-1)
    {
        printf("connect() error\n");
        exit(1);
    }
    close(sockfd);
    return 0;
}
```

```bash
//test.c:
#include <stdio.h>
int main()
{
	int ret = 0;
	long long cnt = 0;
	while(1)
	{
		system("./client 127.0.0.1");
		cnt++;	

		if( 10000 == cnt )
		{
			break;
		}	
	}
	return ret;
}
//运行结果：
./test
./test
pthread-id=870520576,You got a connection from client's ip 127.0.0.1, port 36335 at time 1480235067.980892
3995, 4007, 4006, 3995, 3997 
```

### 结论
只会唤醒一个线程，且每个线程被唤醒的次数基本相等，多个线程监听同一个fd基本没问题
