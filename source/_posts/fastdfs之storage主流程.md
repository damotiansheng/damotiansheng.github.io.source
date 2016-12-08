---
title: fastdfs之storage主流程
date: 2016-12-04 17:08:51
tags: [fastdfs]
---

## 主要工作
[相关文章](http://blog.chinaunix.net/uid-20498361-id-3328762.html)
1）思想：与tracker主流程一样，采用主线程监听得到fd,dispatch子线程（默认4个工作线程）进行处理；
2）创建的线程数：会创建与tracker server数量相等的report thread,用于向tracker server发送相关信息，会创建4个工作线程用于处理任务，会创建一个调度线程，用于执行一些定期任务，如：log_sync_func，fdfs_binlog_sync_func等，还会创建一些IO读写线程，用于读写目录，默认一个目录有一个读线程和一个写线程，线程函数都为dio_thread_entrance函数；
3）主要处理过程：主线程往管道写入得到的连接的相关信息（其实是写入一个fast_task_info地址），工作线程会调用storage_recv_notify_read函数得到该地址，然后依据不同的状态进行不同的处理，具体见storage_recv_notify_read函数；
<!--more-->

## 相关数据结构
```
struct nio_thread_data
{
	struct ioevent_puller ev_puller; //IO multiplexing function: poll, kqueue 
	struct fast_timer timer; // time wheel 
	int pipe_fds[2]; //to wake up current thread who owns current nio_thread_data
	struct fast_task_info *deleted_list;
	ThreadLoopCallback thread_loop_callback;
	void *arg;   //extra argument pointer
};

struct storage_nio_thread_data
{
	struct nio_thread_data thread_data;
	GroupArray group_array;  //FastDHT group array
};
每个工作线程对应一个storage_nio_thread_data结构
```
## 相关代码
```
// fdfs_storaged.c->main函数
int main(int argc, char *argv[])
{
	char *conf_filename;
	int result;
	int sock;
	int wait_count;
	pthread_t schedule_tid;
	struct sigaction act;
	ScheduleEntry scheduleEntries[SCHEDULE_ENTRIES_MAX_COUNT];
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

	log_init2();  //init the log system
	trunk_shared_init(); // init the g_fdfs_base64_context global variable

	conf_filename = argv[1];
	
	if ((result=get_base_path_from_conf_file(conf_filename,
		g_fdfs_base_path, sizeof(g_fdfs_base_path))) != 0) // get base path from conf_filename
	{
		log_destroy();
		return result;
	}

	snprintf(pidFilename, sizeof(pidFilename),
		"%s/data/fdfs_storaged.pid", g_fdfs_base_path);
	
	if ((result=process_action(pidFilename, argv[2], &stop)) != 0)// process command: restart/stop/start
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
		sizeof(g_exe_name)) == NULL) //get the absolute path of exe file saved to g_exe_name
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return errno != 0 ? errno : ENOENT;
	}
#endif

	daemon_init(false); // set to daemon mode 
	umask(0);

	memset(g_bind_addr, 0, sizeof(g_bind_addr));
	
	if ((result=storage_func_init(conf_filename, \
			g_bind_addr, sizeof(g_bind_addr))) != 0) // parse the conf_filename and get some values saved to global variable
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	//*g_bind_addr = '\0', g_server_port=23000
	// socketServer to finish socket, bind, listen
	sock = socketServer(g_bind_addr, g_server_port, &result);
	
	if (sock < 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	// tcpsetserveropt: set send/recv timeout and keepalive parameter
	if ((result=tcpsetserveropt(sock, g_fdfs_network_timeout)) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

	if ((result=write_to_pid_file(pidFilename)) != 0) 
	{
		log_destroy();
		return result;
	}


    // create base_path and the sync path, create file: binlog.000(default), init the sync_thread_lock 
	if ((result=storage_sync_init()) != 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"storage_sync_init fail, program exit!", __LINE__);
		g_continue_flag = false;
		return result;
	}

    // init the lock of reporter_thread_lock, clear g_storage_servers and g_sorted_storages
	if ((result=tracker_report_init()) != 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"tracker_report_init fail, program exit!", __LINE__);
		g_continue_flag = false;
		return result;
	}
	
    //create 4 work thread default, init some lock, build a task object(fast_task_info) pool(g_free_queue)
	if ((result=storage_service_init()) != 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"storage_service_init fail, program exit!", __LINE__);
		g_continue_flag = false;
		return result;
	}

    // init the rand generator
	if ((result=set_rand_seed()) != 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"set_rand_seed fail, program exit!", __LINE__);
		g_continue_flag = false;
		return result;
	}

	memset(&act, 0, sizeof(act));
	sigemptyset(&act.sa_mask);

	act.sa_handler = sigUsrHandler; // sigUsrHandler do nothing
	
	if(sigaction(SIGUSR1, &act, NULL) < 0 || \
		sigaction(SIGUSR2, &act, NULL) < 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"call sigaction fail, errno: %d, error info: %s", \
			__LINE__, errno, STRERROR(errno));
		logCrit("exit abnormally!\n");
		return errno;
	}

	act.sa_handler = sigHupHandler; // set g_log_context.rotate_immediately to true
	
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

	act.sa_handler = sigQuitHandler;
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
		if ((result=storage_httpd_start(g_bind_addr)) != 0)
		{
			logCrit("file: "__FILE__", line: %d, " \
				"storage_httpd_start fail, " \
				"program exit!", __LINE__);
			return result;
		}
	}
#endif

     // create some child thread, a thread conrresoponds to one tracker server
	if ((result=tracker_report_thread_start()) != 0)
	{
		logCrit("file: "__FILE__", line: %d, " \
			"tracker_report_thread_start fail, " \
			"program exit!", __LINE__);
		g_continue_flag = false;
		storage_func_destroy();
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
		g_sync_binlog_buff_interval, fdfs_binlog_sync_func, NULL);
	scheduleArray.count++;

	INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
		scheduleArray.count + 1, TIME_NONE, TIME_NONE, TIME_NONE,
		g_sync_stat_file_interval, fdfs_stat_file_sync_func, NULL);
	scheduleArray.count++;

	if (g_if_use_trunk_file)
	{
		INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
			scheduleArray.count + 1, TIME_NONE, TIME_NONE, TIME_NONE,
			1, trunk_binlog_sync_func, NULL);
		scheduleArray.count++;
	}

	if (g_use_access_log)
	{
		INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
			scheduleArray.count + 1, TIME_NONE, TIME_NONE, TIME_NONE,
			g_sync_log_buff_interval, log_sync_func, &g_access_log_context);
		scheduleArray.count++;

		if (g_rotate_access_log)
		{
			INIT_SCHEDULE_ENTRY_EX(scheduleEntries[scheduleArray.count],
				scheduleArray.count + 1, g_access_log_rotate_time,
				24 * 3600, log_notify_rotate, &g_access_log_context);
			scheduleArray.count++;

			if (g_log_file_keep_days > 0)
			{
				log_set_keep_days(&g_access_log_context,
					g_log_file_keep_days);

				INIT_SCHEDULE_ENTRY(scheduleEntries[scheduleArray.count],
					scheduleArray.count + 1, 1, 0, 0, 24 * 3600,
					log_delete_old_files, &g_access_log_context);
				scheduleArray.count++;
			}
		}
	}

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

     // create a thread to schedule log_sync_func,fdfs_stat_file_sync_func and so on
	if ((result=sched_start(&scheduleArray, &schedule_tid, \
		g_thread_stack_size, (bool * volatile)&g_continue_flag)) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}

     // set current group and user of current process
	if ((result=set_run_by(g_run_by_group, g_run_by_user)) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}
	
    // create read and write thread: dio_thread_entrance for every store_path,
    // one path has a read thread and a write thread default
	if ((result=storage_dio_init()) != 0)
	{
		logCrit("exit abnormally!\n");
		log_destroy();
		return result;
	}
	
	log_set_cache(true); // set g_log_context.log_to_cache to true

	bTerminateFlag = false;
	bAcceptEndFlag = false;
	
	storage_accept_loop(sock); // enter main thread loop, accept to get fd, and wakeup child thread to process
	bAcceptEndFlag = true;

	fdfs_binlog_sync_func(NULL);  //binlog fsync

	if (g_schedule_flag)
	{
		pthread_kill(schedule_tid, SIGINT);
	}

	storage_terminate_threads();  // wakeup work thread to quit
	storage_dio_terminate();      // terminate the io thread

	kill_tracker_report_threads(); // kill report thread
	kill_storage_sync_threads();

	wait_count = 0;
	while (g_storage_thread_count != 0 || \
		g_dio_thread_count != 0 || \
		g_tracker_reporter_count > 0 || \
		g_schedule_flag)
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
		if (++wait_count > 6000)
		{
			logWarning("waiting timeout, exit!");
			break;
		}
	}

	tracker_report_destroy();
	storage_service_destroy();
	storage_sync_destroy();
	storage_func_destroy();

	if (g_if_use_trunk_file)
	{
		trunk_sync_destroy();
		storage_trunk_destroy();
	}

	logInfo("exit normally.\n");
	log_destroy();
	
	delete_pid_file(pidFilename);
	return 0;
}

```
```
// 主线程函数
// storage server main thread, accept to get the fd, save some info, and write the address of (struct fast_task_info),
// it contains the fd
static void *accept_thread_entrance(void* arg)
{
	int server_sock;
	int incomesock;
	struct sockaddr_in inaddr;
	socklen_t sockaddr_len;
	in_addr_t client_addr;
	char szClientIp[IP_ADDRESS_SIZE];
	long task_addr;
	struct fast_task_info *pTask;
	StorageClientInfo *pClientInfo;
	struct storage_nio_thread_data *pThreadData;

	server_sock = (long)arg;
	
	while (g_continue_flag)
	{
		sockaddr_len = sizeof(inaddr);
		incomesock = accept(server_sock, (struct sockaddr*)&inaddr, \
					&sockaddr_len);
		
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

		client_addr = getPeerIpaddr(incomesock, \
				szClientIp, IP_ADDRESS_SIZE);
		
		if (g_allow_ip_count >= 0)
		{
			if (bsearch(&client_addr, g_allow_ip_addrs, \
					g_allow_ip_count, sizeof(in_addr_t), \
					cmp_by_ip_addr_t) == NULL)
			{
				logError("file: "__FILE__", line: %d, " \
					"ip addr %s is not allowed to access", \
					__LINE__, szClientIp);

				close(incomesock);
				continue;
			}
		}

        // set non block for fd
		if (tcpsetnonblockopt(incomesock) != 0)
		{
			close(incomesock);
			continue;
		}

		pTask = free_queue_pop(); // get a free task object from free task object pool(g_free_queue)
		
		if (pTask == NULL)
		{
			logError("file: "__FILE__", line: %d, " \
				"malloc task buff failed", \
				__LINE__);
			close(incomesock);
			continue;
		}

		pClientInfo = (StorageClientInfo *)pTask->arg; // pTask->arg is pointer to (StorageClientInfo *), see storage_service_init->free_queue_init_ex func
		pTask->event.fd = incomesock;
		pClientInfo->stage = FDFS_STORAGE_STAGE_NIO_INIT; //will be used in storage_recv_notify_read function
		pClientInfo->nio_thread_index = pTask->event.fd % g_work_threads;
		pThreadData = g_nio_thread_data + pClientInfo->nio_thread_index;

		strcpy(pTask->client_ip, szClientIp);

		task_addr = (long)pTask; // convert address to long, and write the long
		if (write(pThreadData->thread_data.pipe_fds[1], &task_addr, \
			sizeof(task_addr)) != sizeof(task_addr))
		{
			close(incomesock);
			free_queue_push(pTask);
			logError("file: "__FILE__", line: %d, " \
				"call write failed, " \
				"errno: %d, error info: %s", \
				__LINE__, errno, STRERROR(errno));
		}
        else
        {
            int current_connections;
            current_connections = __sync_add_and_fetch(&g_storage_stat.connection.
                    current_count, 1);
			
            if (current_connections > g_storage_stat.connection.max_count) 
			{
                g_storage_stat.connection.max_count = current_connections;
            }
			
            ++g_stat_change_count;
        }
	}

	return NULL;
}

//工作线程函数
// default is 4 threads to call work_thread_entrance
static void *work_thread_entrance(void* arg)
{
	int result;
	struct storage_nio_thread_data *pThreadData;

	pThreadData = (struct storage_nio_thread_data *)arg;
	
	if (g_check_file_duplicate) // check duplicate, here just leave it
	{
		if ((result=fdht_copy_group_array(&(pThreadData->group_array),\
				&g_group_array)) != 0)
		{
			pthread_mutex_lock(&g_storage_thread_lock);
			g_storage_thread_count--;
			pthread_mutex_unlock(&g_storage_thread_lock);
			return NULL;
		}
	}
	
	ioevent_loop(&pThreadData->thread_data, storage_recv_notify_read,
		task_finish_clean_up, &g_continue_flag);  // epoll wait to listen

	// free free(ioevent->events) and close(ioevent->poll_fd) 
	ioevent_destroy(&pThreadData->thread_data.ev_puller);

	if (g_check_file_duplicate)
	{
		if (g_keep_alive)
		{
			fdht_disconnect_all_servers(&(pThreadData->group_array));
		}

		fdht_free_group_array(&(pThreadData->group_array));
	}

	if ((result=pthread_mutex_lock(&g_storage_thread_lock)) != 0)
	{
		logError("file: "__FILE__", line: %d, " \
			"call pthread_mutex_lock fail, " \
			"errno: %d, error info: %s", \
			__LINE__, result, STRERROR(result));
	}
	
	g_storage_thread_count--;
	
	if ((result=pthread_mutex_unlock(&g_storage_thread_lock)) != 0)
	{
		logError("file: "__FILE__", line: %d, " \
			"call pthread_mutex_lock fail, " \
			"errno: %d, error info: %s", \
			__LINE__, result, STRERROR(result));
	}

	logDebug("file: "__FILE__", line: %d, " \
		"nio thread exited, thread count: %d", \
		__LINE__, g_storage_thread_count);

	return NULL;
}

void storage_recv_notify_read(int sock, short event, void *arg)
{
	struct fast_task_info *pTask;
	StorageClientInfo *pClientInfo;
	long task_addr;
	int64_t remain_bytes;
	int bytes;
	int result;

	while (1)
	{         // read a address of buffer, it is the address of (struct fast_task_info), see accept_thread_entrance func
		if ((bytes=read(sock, &task_addr, sizeof(task_addr))) < 0)
		{
			if (!(errno == EAGAIN || errno == EWOULDBLOCK))
			{
				logError("file: "__FILE__", line: %d, " \
					"call read failed, " \
					"errno: %d, error info: %s", \
					__LINE__, errno, STRERROR(errno));
			}

			break;
		}
		else if (bytes == 0)
		{
			logError("file: "__FILE__", line: %d, " \
				"call read failed, end of file", __LINE__);
			break;
		}

		pTask = (struct fast_task_info *)task_addr; 
		pClientInfo = (StorageClientInfo *)pTask->arg;

		if (pTask->event.fd < 0)  //quit flag
		{
			return;
		}

		/* //logInfo("=====thread index: %d, pTask->event.fd=%d", \
			pClientInfo->nio_thread_index, pTask->event.fd);
		*/

		if (pClientInfo->stage & FDFS_STORAGE_STAGE_DIO_THREAD)
		{
			pClientInfo->stage &= ~FDFS_STORAGE_STAGE_DIO_THREAD;
		}
		
		switch (pClientInfo->stage)
		{
			case FDFS_STORAGE_STAGE_NIO_INIT:
				result = storage_nio_init(pTask);
				break;
				
			case FDFS_STORAGE_STAGE_NIO_RECV:
				pTask->offset = 0;
				remain_bytes = pClientInfo->total_length - \
					       pClientInfo->total_offset;
				
				if (remain_bytes > pTask->size)
				{
					pTask->length = pTask->size;
				}
				else
				{
					pTask->length = remain_bytes;
				}

				if (set_recv_event(pTask) == 0)
				{
					client_sock_read(pTask->event.fd,
						IOEVENT_READ, pTask);
				}
				
				result = 0;
				break;
				
			case FDFS_STORAGE_STAGE_NIO_SEND:
				result = storage_send_add_event(pTask);
				break;
			case FDFS_STORAGE_STAGE_NIO_CLOSE:
				result = EIO;   //close this socket
				break;
			default:
				logError("file: "__FILE__", line: %d, " \
					"invalid stage: %d", __LINE__, \
					pClientInfo->stage);
				result = EINVAL;
				break;
		}

		if (result != 0)
		{
			add_to_deleted_list(pTask);
		}
	}
}

```
