---
title: fastdfs之tracker主流程
date: 2016-12-04 13:01:50
tags: [fastdfs]
---

## 思想
[相关文章1](http://blog.chinaunix.net/uid-20498361-id-3328763.html)
[相关文章2](http://blog.csdn.net/makamus/article/details/13759545)
主要思想是采用主线程监听并accept连接（代码中tracker_accept_loop函数可以依据全局变量g_accept_threads而开启多个子线程进行监听，默认只有一个主线程监听），得到fd,然后将该fd写入到某一个管道，而子线程（工作线程）（默认4个）监听到管道的另外一端，其处理函数为recv_notify_read，该函数读取管道得到该fd的值，再次将它加入到某个线程（不一定是当前线程）的监听事件中去，处理函数为client_sock_read，该函数当读完客户端完整的请求数据包时会调用tracker_deal_task函数进行处理，当需要进行写时会调用client_sock_write函数进行处理，这样主线程监听得到连接，交给子线程进行处理；
<!--more-->

## 源码
```
相关数据结构
struct nio_thread_data
{
	struct ioevent_puller ev_puller; //IO multiplexing function: poll, kqueue 
	struct fast_timer timer; // time wheel 
	int pipe_fds[2]; //to wake up current thread who owns current nio_thread_data
	struct fast_task_info *deleted_list;
	ThreadLoopCallback thread_loop_callback;
	void *arg;   //extra argument pointer
};
每个工作线程对应一个该结构体，以实现one thread, one loop,
```

```
//fdfs_trackerd.c->main函数
int main(int argc, char *argv[])
{
	char *conf_filename;
	int result;
	int wait_count;
	int sock;
	pthread_t schedule_tid;
	struct sigaction act;
	ScheduleEntry scheduleEntries[SCHEDULE_ENTRIES_COUNT]; //保存需要定时执行的任务
	ScheduleArray scheduleArray;
	char pidFilename[MAX_PATH_SIZE];
	bool stop;

	if (argc < 2)
	{
		usage(argv[0]);
		return 1;
	}

	g_current_time = time(NULL);
	g_up_time = g_current_time;
	srand(g_up_time);

	log_init2();

	conf_filename = argv[1];
	
	if ((result=get_base_path_from_conf_file(conf_filename,
		g_fdfs_base_path, sizeof(g_fdfs_base_path))) != 0)  //从配置文件中得到base path
	{
		log_destroy();
		return result;
	}

	snprintf(pidFilename, sizeof(pidFilename),
		"%s/data/fdfs_trackerd.pid", g_fdfs_base_path);
	
	if ((result=process_action(pidFilename, argv[2], &stop)) != 0) //处理命令： fdfs_trackerd restart/stop/start
	{
		if (result == EINVAL)
		{
			usage(argv[0]);
		}
		
		log_destroy();
		return result;
	}
	
	if (stop)
	{
		log_destroy();
		return 0;
	}

#if defined(DEBUG_FLAG) && defined(OS_LINUX)
	if (getExeAbsoluteFilename(argv[0], g_exe_name, \
		sizeof(g_exe_name)) == NULL)   //得到可执行文件的绝对路径，保存到g_exe_name
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return errno != 0 ? errno : ENOENT;
	}
#endif

	memset(bind_addr, 0, sizeof(bind_addr));

	if ((result=tracker_load_from_conf_file(conf_filename, \
			bind_addr, sizeof(bind_addr))) != 0)       //得到监听的地址
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

    
	//	load last status of tracker last time from file: base_path/data/.tracker_status
	if ((result=tracker_load_status_from_file(&g_tracker_last_status)) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

    // init the global variable: g_base64_context
	base64_init_ex(&g_base64_context, 0, '-', '_', '.');

	// set a rand num
	if ((result=set_rand_seed()) != 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"set_rand_seed fail, program exit!", __LINE__);
		return result;
	}

	if ((result=tracker_mem_init()) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	sock = socketServer(bind_addr, g_server_port, &result);
	
	if (sock < 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

    // set send/recv timeout and keepalive parameter
	if ((result=tcpsetserveropt(sock, g_fdfs_network_timeout)) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	daemon_init(false); //后台进程
	umask(0);
	
	if ((result=write_to_pid_file(pidFilename)) != 0)
	{
		log_destroy();
		return result;
	}

    // create work thread: work_thread_entrance func
    // every thread has its event loop
	if ((result=tracker_service_init()) != 0)  //创建工作线程，每个工作线程对应一个nio_thread_data结构体
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	
	memset(&act, 0, sizeof(act));
	sigemptyset(&act.sa_mask);
   
	act.sa_handler = sigUsrHandler;
	if(sigaction(SIGUSR1, &act, NULL) < 0 || \
		sigaction(SIGUSR2, &act, NULL) < 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"call sigaction fail, errno: %d, error info: %s", \
			__LINE__, errno, STRERROR(errno));
		logCrit("exit abnormally!\n");
		return errno;
	}

	act.sa_handler = sigHupHandler;
	if(sigaction(SIGHUP, &act, NULL) < 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"call sigaction fail, errno: %d, error info: %s", \
			__LINE__, errno, STRERROR(errno));
		logCrit("exit abnormally!\n");
		return errno;
	}
	
	act.sa_handler = SIG_IGN;
	if(sigaction(SIGPIPE, &act, NULL) < 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"call sigaction fail, errno: %d, error info: %s", \
			__LINE__, errno, STRERROR(errno));
		logCrit("exit abnormally!\n");
		return errno;
	}

	act.sa_handler = sigQuitHandler;  // send FDFS_PROTO_CMD_QUIT command when caught SIGINT,SIGTERM,SIGQUIT
	if(sigaction(SIGINT, &act, NULL) < 0 || \
		sigaction(SIGTERM, &act, NULL) < 0 || \
		sigaction(SIGQUIT, &act, NULL) < 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"call sigaction fail, errno: %d, error info: %s", \
			__LINE__, errno, STRERROR(errno));
		logCrit("exit abnormally!\n");
		return errno;
	}

#if defined(DEBUG_FLAG)
/*
#if defined(OS_LINUX)
	memset(&act, 0, sizeof(act));
	sigemptyset(&act.sa_mask);
        act.sa_sigaction = sigSegvHandler;
        act.sa_flags = SA_SIGINFO;
        if (sigaction(SIGSEGV, &act, NULL) < 0 || \
        	sigaction(SIGABRT, &act, NULL) < 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"call sigaction fail, errno: %d, error info: %s", \
			__LINE__, errno, STRERROR(errno));
		logCrit("exit abnormally!\n");
		return errno;
	}
#endif
*/

	memset(&act, 0, sizeof(act));
	sigemptyset(&act.sa_mask);
	act.sa_handler = sigDumpHandler; 
	if(sigaction(SIGUSR1, &act, NULL) < 0 || \
		sigaction(SIGUSR2, &act, NULL) < 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"call sigaction fail, errno: %d, error info: %s", \
			__LINE__, errno, STRERROR(errno));
		logCrit("exit abnormally!\n");
		return errno;
	}
#endif

#ifdef WITH_HTTPD
	if (!g_http_params.disabled)
	{
		if ((result=tracker_httpd_start(bind_addr)) != 0)
		{
			logCrit("file: "__FILE__", line: %d, " \
				"tracker_httpd_start fail, program exit!", \
				__LINE__);
			return result;
		}

	}

	if ((result=tracker_http_check_start()) != 0)   
	{
		logCrit("file: "__FILE__", line: %d, " \
			"tracker_http_check_start fail, " \
			"program exit!", __LINE__);
		return result;
	}
#endif

	if ((result=set_run_by(g_run_by_group, g_run_by_user)) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	scheduleArray.entries = scheduleEntries;
	scheduleArray.count = 0;
	memset(scheduleEntries, 0, sizeof(scheduleEntries));

	INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
		scheduleArray.count + 1, TIME_NONE, TIME_NONE, TIME_NONE,
		g_sync_log_buff_interval, log_sync_func, &g_log_context);
	scheduleArray.count++;

	INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
		scheduleArray.count + 1, TIME_NONE, TIME_NONE, TIME_NONE,
		g_check_active_interval, tracker_mem_check_alive, NULL);
	scheduleArray.count++;

	INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
		scheduleArray.count + 1, 0, 0, 0,
		TRACKER_SYNC_STATUS_FILE_INTERVAL,
		tracker_write_status_to_file, NULL);
	scheduleArray.count++;

	if (g_rotate_error_log)
	{
		INIT_SCHEDULE_ENTRY_EX(scheduleEntries[scheduleArray.count],
			scheduleArray.count + 1, g_error_log_rotate_time,
			24 * 3600, log_notify_rotate, &g_log_context);
		scheduleArray.count++;

		if (g_log_file_keep_days > 0)
		{
			log_set_keep_days(&g_log_context, g_log_file_keep_days);

			INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
				scheduleArray.count + 1, 1, 0, 0, 24 * 3600,
				log_delete_old_files, &g_log_context);
			scheduleArray.count++;
		}
	}

    // create a thread to execute some periodic task above, such as:log_sync_func,tracker_mem_check_alive
    // tracker_write_status_to_file and so on
	if ((result=sched_start(&scheduleArray, &schedule_tid, \
		g_thread_stack_size, (bool * volatile)&g_continue_flag)) != 0) //创建调度线程，以定时执行上面的任务：log_sync_func，tracker_mem_check_alive等
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	if ((result=tracker_relationship_init()) != 0) //初始化主lead
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	log_set_cache(true); //set g_log_context.log_to_cache to true

	bTerminateFlag = false;
	bAcceptEndFlag = false;

	tracker_accept_loop(sock);  // the sock is tracker server socket，主线程循环，得到连接，dispatch子线程处理
	bAcceptEndFlag = true;
	
	if (g_schedule_flag)
	{
		pthread_kill(schedule_tid, SIGINT);
	}
	
	tracker_terminate_threads(); 

#ifdef WITH_HTTPD
	if (g_http_check_flag)
	{
		tracker_http_check_stop();
	}

	while (g_http_check_flag)
	{
		usleep(50000);
	}
#endif

	wait_count = 0;

    // g_schedule_flag is true means the schedule thread is not ended, see sched_thread_entrance
	while ((g_tracker_thread_count != 0) || g_schedule_flag)
	{

/*
#if defined(DEBUG_FLAG) && defined(OS_LINUX)
		if (bSegmentFault)
		{
			sleep(5);
			break;
		}
#endif
*/

		usleep(10000);

		if (++wait_count > 3000)
		{
			logWarning("waiting timeout, exit!");
			break;
		}
	}
	
	tracker_mem_destroy(); // free the memory in g_groups and free lock allocated previously
	tracker_service_destroy(); // wait all child thread to exit
	tracker_relationship_destroy(); // do nothing
	
	logInfo("exit normally.\n");
	log_destroy(); // destory the log system
	
	delete_pid_file(pidFilename);
	return 0;
}
```

```
//初始化工作线程
// create work thread: work_thread_entrance func
// http://slucx.blog.chinaunix.net/uid-29504236-id-4556391.html
// http://slucx.blog.chinaunix.net/uid-29504236-id-4556487.html
int tracker_service_init()
{
#define ALLOC_CONNECTIONS_ONCE 1024
	int result;
	int bytes;
    int init_connections;
	struct nio_thread_data *pThreadData;
	struct nio_thread_data *pDataEnd;
	pthread_t tid;
	pthread_attr_t thread_attr;

	if ((result=init_pthread_lock(&tracker_thread_lock)) != 0)
	{
		return result;
	}

	if ((result=init_pthread_lock(&lb_thread_lock)) != 0)
	{
		return result;
	}

	if ((result=init_pthread_attr(&thread_attr, g_thread_stack_size)) != 0)
	{
		logError("file: "__FILE__", line: %d, " \
			"init_pthread_attr fail, program exit!", __LINE__);
		return result;
	}

        init_connections = g_max_connections < ALLOC_CONNECTIONS_ONCE ?
        g_max_connections : ALLOC_CONNECTIONS_ONCE;
	
	if ((result=free_queue_init_ex(g_max_connections, init_connections,
                    ALLOC_CONNECTIONS_ONCE, TRACKER_MAX_PACKAGE_SIZE,
                    TRACKER_MAX_PACKAGE_SIZE, sizeof(TrackerClientInfo))) != 0)
	{
		return result;
	}

	// every thread has one nio_thread_data
	bytes = sizeof(struct nio_thread_data) * g_work_threads;
	g_thread_data = (struct nio_thread_data *)malloc(bytes );
	
	if (g_thread_data == NULL)
	{
		logError("file: "__FILE__", line: %d, " \
			"malloc %d bytes fail, errno: %d, error info: %s", \
			__LINE__, bytes, errno, STRERROR(errno));
		return errno != 0 ? errno : ENOMEM;
	}
	
	memset(g_thread_data, 0, bytes);

	g_tracker_thread_count = 0;
	pDataEnd = g_thread_data + g_work_threads;   // 默认g_work_threads为4
	
	for (pThreadData=g_thread_data; pThreadData<pDataEnd; pThreadData++)
	{
		if (ioevent_init(&pThreadData->ev_puller,
			g_max_connections + 2, 1000, 0) != 0) // g_max_connections=256，初始化IO复用函数，超时为1秒
		{
			result  = errno != 0 ? errno : ENOMEM;
			logError("file: "__FILE__", line: %d, " \
				"ioevent_init fail, " \
				"errno: %d, error info: %s", \
				__LINE__, result, STRERROR(result));
			return result;
		}

		result = fast_timer_init(&pThreadData->timer,
				2 * g_fdfs_network_timeout, g_current_time); //初始化超时事件time wheel
		
		if (result != 0)
		{
			logError("file: "__FILE__", line: %d, " \
				"fast_timer_init fail, " \
				"errno: %d, error info: %s", \
				__LINE__, result, STRERROR(result));
			return result;
		}

		if (pipe(pThreadData->pipe_fds) != 0)  //创建管道
		{
			result = errno != 0 ? errno : EPERM;
			logError("file: "__FILE__", line: %d, " \
				"call pipe fail, " \
				"errno: %d, error info: %s", \
				__LINE__, result, STRERROR(result));
			break;
		}

#if defined(OS_LINUX)
		if ((result=fd_add_flags(pThreadData->pipe_fds[0], \
				O_NONBLOCK | O_NOATIME)) != 0) // O_NOATIME: Do not update the file last access time 
		{
			break;
		}
#else
		if ((result=fd_add_flags(pThreadData->pipe_fds[0], \
				O_NONBLOCK)) != 0)
		{
			break;
		}
#endif

		if ((result=pthread_create(&tid, &thread_attr, \
			work_thread_entrance, pThreadData)) != 0)  //创建工作线程
		{
			logError("file: "__FILE__", line: %d, " \
				"create thread failed, startup threads: %d, " \
				"errno: %d, error info: %s", \
				__LINE__, g_tracker_thread_count, \
				result, STRERROR(result));
			break;
		}
		else
		{
			if ((result=pthread_mutex_lock(&tracker_thread_lock)) != 0)
			{
				logError("file: "__FILE__", line: %d, " \
					"call pthread_mutex_lock fail, " \
					"errno: %d, error info: %s", \
					__LINE__, result, STRERROR(result));
			}
			
			g_tracker_thread_count++; //工作线程数量递增
			
			if ((result=pthread_mutex_unlock(&tracker_thread_lock)) != 0)
			{
				logError("file: "__FILE__", line: %d, " \
					"call pthread_mutex_lock fail, " \
					"errno: %d, error info: %s", \
					__LINE__, result, STRERROR(result));
			}
		}
		
	}

	pthread_attr_destroy(&thread_attr);

	return 0;
}

//工作线程work_thread_entrance
// tracker child thread
static void *work_thread_entrance(void* arg)
{
	int result;
	struct nio_thread_data *pThreadData;

	pThreadData = (struct nio_thread_data *)arg;
	
	ioevent_loop(pThreadData, recv_notify_read, task_finish_clean_up,
		&g_continue_flag);  // 循环等待事件的到来并处理
	
	ioevent_destroy(&pThreadData->ev_puller);

	if ((result=pthread_mutex_lock(&tracker_thread_lock)) != 0)
	{
		logError("file: "__FILE__", line: %d, " \
			"call pthread_mutex_lock fail, " \
			"errno: %d, error info: %s", \
			__LINE__, result, STRERROR(result));
	}
	
	g_tracker_thread_count--;  //工作线程数量递减
	
	if ((result=pthread_mutex_unlock(&tracker_thread_lock)) != 0)
	{
		logError("file: "__FILE__", line: %d, " \
			"call pthread_mutex_lock fail, " \
			"errno: %d, error info: %s", \
			__LINE__, result, STRERROR(result));
	}

	return NULL;
}
```

```
//主线程
// accept a client request and dispatch request to child thread for processing
static void *accept_thread_entrance(void* arg)
{
	int server_sock;
	int incomesock;
	struct sockaddr_in inaddr;
	socklen_t sockaddr_len;
	struct nio_thread_data *pThreadData;
	server_sock = (long)arg;
	
	while (g_continue_flag)
	{
		sockaddr_len = sizeof(inaddr);
		incomesock = accept(server_sock, (struct sockaddr*)&inaddr, &sockaddr_len);
		
		if (incomesock < 0) //error
		{
			if (!(errno == EINTR || errno == EAGAIN))
			{
				logError("file: "__FILE__", line: %d, " \
					"accept failed, " \
					"errno: %d, error info: %s", \
					__LINE__, errno, STRERROR(errno));
			}

			continue;
		}

		pThreadData = g_thread_data + incomesock % g_work_threads;
		
		if (write(pThreadData->pipe_fds[1], &incomesock, \
			sizeof(incomesock)) != sizeof(incomesock)) // 将得到的client fd写入某个管道中，wakeup child thread
		{
			close(incomesock);
			logError("file: "__FILE__", line: %d, " \
				"call write failed, " \
				"errno: %d, error info: %s", \
				__LINE__, errno, STRERROR(errno));
		}
        else
        {
            int current_connections;
            current_connections = __sync_add_and_fetch(&g_connection_stat.
                    current_count, 1);
			
            if (current_connections > g_connection_stat.max_count) 
			{
                g_connection_stat.max_count = current_connections;
            }
			
        }
	}

	return NULL;
}
```
