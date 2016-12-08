---
title: fastdfs之事件监听
date: 2016-12-04 14:25:51
tags: [fastdfs,epoll]
---

## 思想
fastdfs需要监听某个fd的事件，当fd就绪时自动调用某个函数进行处理，其思想是将该事件的回调函数，fd保存到epoll_event结构体中，当该fd就绪时就可得到该结构体，从而调用该回调函数；
[epoll机制](http://blog.csdn.net/yusiguyuan/article/details/15027821)
<!--more-->

## 相关数据结构
```
typedef struct ioevent_puller {
    int size;  //set to g_max_connections + 2,default is 256+2, max events (fd) , equal as the parameter of epoll_create
    int extra_events;
    int poll_fd; // poll_fd = epoll_create(ioevent->size)

/*
When  successful,  epoll_wait()  returns the number of file descriptors
       ready for the requested I/O, or zero if no file descriptor became ready
       during  the  requested  timeout  milliseconds.   When  an error occurs,
       epoll_wait() returns -1 and errno is set appropriately.
*/
    struct 
	{
        int index;
        int count;  // count=epoll_wait(...)
    } iterator;  //for deal event loop

#if IOEVENT_USE_EPOLL
    struct epoll_event *events; // the size is ( size * sizeof(struct epoll_event) )
    int timeout;                  // default is 1000
#elif IOEVENT_USE_KQUEUE
    struct kevent *events;
    struct timespec timeout;
    int care_events;
#elif IOEVENT_USE_PORT
    port_event_t *events;
    timespec_t timeout;
#endif
} IOEventPoller;

typedef void (*IOEventCallback) (int sock, short event, void *arg);


typedef struct ioevent_entry
{
	int fd;                // it will be added to nio_thread_data->ev_puller
	FastTimerEntry timer; // for event timeout, it will be added to nio_thread_data->timer
	IOEventCallback callback;
} IOEventEntry;

#if IOEVENT_USE_EPOLL
  #define IOEVENT_GET_EVENTS(ioevent, index) \
      ioevent->events[index].events

#if IOEVENT_USE_EPOLL
  #define IOEVENT_GET_DATA(ioevent, index)  \
      ioevent->events[index].data.ptr

```

```
//初始化
//ioevent_init(&pThreadData->ev_puller,g_max_connections + 2, 1000, 0) // 1000毫秒

// size=256+2, timeout_ms=1000, extra_events=0
// ioevent_init(&pThreadData->ev_puller, g_max_connections + 2, 1000, 0)
int ioevent_init(IOEventPoller *ioevent, const int size,
    const int timeout_ms, const int extra_events)
{
  int bytes;

  ioevent->size = size;
  ioevent->extra_events = extra_events;
  ioevent->iterator.index = 0;
  ioevent->iterator.count = 0;

#if IOEVENT_USE_EPOLL
  ioevent->poll_fd = epoll_create(ioevent->size);
  bytes = sizeof(struct epoll_event) * size;
  ioevent->events = (struct epoll_event *)malloc(bytes);
#elif IOEVENT_USE_KQUEUE
  ioevent->poll_fd = kqueue();
  bytes = sizeof(struct kevent) * size;
  ioevent->events = (struct kevent *)malloc(bytes);
  ioevent->care_events = 0;
#elif IOEVENT_USE_PORT
  ioevent->poll_fd = port_create();
  bytes = sizeof(port_event_t) * size;
  ioevent->events = (port_event_t *)malloc(bytes);
#endif

  if (ioevent->events == NULL) 
  {
    return errno != 0 ? errno : ENOMEM;
  }

  // set the ioevent->timeout to timeout_ms
  ioevent_set_timeout(ioevent, timeout_ms); // 设置超时时间为1秒

  return 0;
}


//事件监听处理循环
//ioevent_loop(pThreadData, recv_notify_read, task_finish_clean_up, &g_continue_flag);
/*
ioevent_loop(pThreadData, recv_notify_read, task_finish_clean_up,
		&g_continue_flag);
		// g_continue_flag is true default
*/
int ioevent_loop(struct nio_thread_data *pThreadData,
	IOEventCallback recv_notify_callback, TaskCleanUpCallback
	clean_up_callback, volatile bool *continue_flag)
{
	int result;
	IOEventEntry ev_notify;
	FastTimerEntry head;
	struct fast_task_info *pTask;
	time_t last_check_time;
	int count;

	memset(&ev_notify, 0, sizeof(ev_notify));
	ev_notify.fd = pThreadData->pipe_fds[0];
	ev_notify.callback = recv_notify_callback;
	
	if (ioevent_attach(&pThreadData->ev_puller,
		pThreadData->pipe_fds[0], IOEVENT_READ,
		&ev_notify) != 0)
	{
		result = errno != 0 ? errno : ENOMEM;
		logCrit("file: "__FILE__", line: %d, " \
			"ioevent_attach fail, " \
			"errno: %d, error info: %s", \
			__LINE__, result, STRERROR(result));
		return result;
	}

    pThreadData->deleted_list = NULL;
	last_check_time = g_current_time;
	
	while (*continue_flag)
	{
		// one seconds later, ioevent_poll will return
		pThreadData->ev_puller.iterator.count = ioevent_poll(&pThreadData->ev_puller);
		
		if (pThreadData->ev_puller.iterator.count > 0)
		{
			deal_ioevents(&pThreadData->ev_puller);
		}
		else if (pThreadData->ev_puller.iterator.count < 0) // error occured
		{
			result = errno != 0 ? errno : EINVAL;
			
			if (result != EINTR)
			{
				logError("file: "__FILE__", line: %d, " \
					"ioevent_poll fail, " \
					"errno: %d, error info: %s", \
					__LINE__, result, STRERROR(result));
				return result;
			}
		}

        // timeout, 1 second later 
		if (pThreadData->deleted_list != NULL) // cleanup task callback is not null
		{
			count = 0;
			
			while (pThreadData->deleted_list != NULL)
			{
				pTask = pThreadData->deleted_list;
				pThreadData->deleted_list = pTask->next;

				clean_up_callback(pTask);
				count++;
			}
			
			logDebug("cleanup task count: %d", count);
		}

		if (g_current_time - last_check_time > 0)
		{
			last_check_time = g_current_time;  // the unit of g_current_time is seconds 
			count = fast_timer_timeouts_get(
				&pThreadData->timer, g_current_time, &head); // get the expire event count
				
			if (count > 0)  // timeout event has been saved to head
			{
				deal_timeouts(&head); // process the timeout event
			}
		}

        if (pThreadData->thread_loop_callback != NULL) 
		{
            pThreadData->thread_loop_callback(pThreadData);  // call this function every one loop 
        }
		
	}

	return 0;
}


int ioevent_poll(IOEventPoller *ioevent)
{
#if IOEVENT_USE_EPOLL
  return epoll_wait(ioevent->poll_fd, ioevent->events, ioevent->size, ioevent->timeout);
#elif IOEVENT_USE_KQUEUE
  return kevent(ioevent->poll_fd, NULL, 0, ioevent->events, ioevent->size, &ioevent->timeout);
#elif IOEVENT_USE_PORT
  int result;
  int retval;
  unsigned int nget = 1;
  if((retval = port_getn(ioevent->poll_fd, ioevent->events,
          ioevent->size, &nget, &ioevent->timeout)) == 0)
  {
    result = (int)nget;
  } else {
    switch(errno) {
      case EINTR:
      case EAGAIN:
      case ETIME:
        if (nget > 0) {
          result = (int)nget;
        }
        else {
          result = 0;
        }
        break;
      default:
        result = -1;
        break;
    }
  }
  return result;
#else
#error port me
#endif
}

//add a fd event to ioevent to listen
// ioevent_attach(&pThread->ev_puller,sock, event, pTask)
int ioevent_attach(IOEventPoller *ioevent, const int fd, const int e,
    void *data)
{
#if IOEVENT_USE_EPOLL
  struct epoll_event ev;
  memset(&ev, 0, sizeof(ev));
  ev.events = e | ioevent->extra_events;
  ev.data.ptr = data;// 将参数保存到data.ptr中去，依据上面的调用data为IOEventEntry结构类型
  return epoll_ctl(ioevent->poll_fd, EPOLL_CTL_ADD, fd, &ev);
#elif IOEVENT_USE_KQUEUE
  struct kevent ev[2];
  int n = 0;
  if (e & IOEVENT_READ) {
    EV_SET(&ev[n++], fd, EVFILT_READ, EV_ADD | ioevent->extra_events, 0, 0, data);
  }
  if (e & IOEVENT_WRITE) {
    EV_SET(&ev[n++], fd, EVFILT_WRITE, EV_ADD | ioevent->extra_events, 0, 0, data);
  }
  ioevent->care_events = e;
  return kevent(ioevent->poll_fd, ev, n, NULL, 0, NULL);
#elif IOEVENT_USE_PORT
  return port_associate(ioevent->poll_fd, PORT_SOURCE_FD, fd, e, data);
#endif
}

// 事件处理
// process the io events by callback function 
// struct epoll_event: http://blog.csdn.net/wangrice2004/article/details/6651320 
static void deal_ioevents(IOEventPoller *ioevent)
{
	int event;
	IOEventEntry *pEntry;

	for (ioevent->iterator.index=0; ioevent->iterator.index < ioevent->iterator.
            count; ioevent->iterator.index++)
	{
		event = IOEVENT_GET_EVENTS(ioevent, ioevent->iterator.index);
		pEntry = (IOEventEntry *)IOEVENT_GET_DATA(ioevent, ioevent->iterator.index);  // 得到ioevent_attach设置的data，为IOEventEntry类型
		// pEntry is either IOEventEntry(ioevent_loop -> ioevent_attach)or fast_task_info(ioevent_set->ioevent_attach)
		// but the first elem of fast_task_info is also IOEventEntry
		
        if (pEntry != NULL)  // call the callback function, we set data in ioevent_attach func
	{
            pEntry->callback(pEntry->fd, event, pEntry->timer.data); // pEntry->timer.data is fast_task_info(see ioevent_set func)
        }
        else 
		{
            logDebug("file: "__FILE__", line: %d, "
                    "ignore iovent : %d, index: %d", __LINE__, event, ioevent->iterator.index);
        }
		
	}
}

```
