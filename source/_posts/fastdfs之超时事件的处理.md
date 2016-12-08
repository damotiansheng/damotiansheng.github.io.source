---
title: fastdfs之超时事件的处理
date: 2016-12-04 14:52:34
tags: [fastdfs,超时事件]
---


## 思想
fastdfs有这样的一个需求，设置一个超时时间，时间到了，自动调用某个函数，其实现的思想有点像Timing wheel的思想，构造一个60（默认）个格子的循环队列，插入超时事件时，插入到(该事件超时时间-循环队列基准时间)%60的对应格子上去，每个格子是一个双向链表，接着采用epoll IO复用函数，1秒后，epoll超时返回，然后依据循环队列中的当前时间，遍历该循环队列，直到循环队列中的当前时间等于当前系统时间，遍历时，判断事件是否超时，将所有已经超时的事件提取出来（从循环队列中删除），构造成链表，接着遍历该链表，依次调用超时事件的回调函数，达到超时事件的处理；
[Timing wheel1](http://blog.csdn.net/solstice/article/details/6395098)
[Timing wheel2](http://blog.csdn.net/mindfloating/article/details/8033340)
<!--more-->

## 相关数据结构
```
typedef struct fast_timer_entry {
  int64_t expires;
  void *data;
  struct fast_timer_entry *prev;
  struct fast_timer_entry *next;
  bool rehash;
} FastTimerEntry;  // pre and next to construct a double linked list

typedef struct fast_timer_slot {
  struct fast_timer_entry head;
} FastTimerSlot;


typedef struct fast_timer 
{
  int slot_count;    //time wheel slot count, default is 60
  int64_t base_time; //base time for slot 0, set to current time， 基准事件
  int64_t current_time; //set to current time
  FastTimerSlot *slots; //the size is sizeof(FastTimerSlot) * slot_count
} FastTimer;

#define TIMER_GET_SLOT_INDEX(timer, expires) \
  (((expires) - timer->base_time) % timer->slot_count)

#define TIMER_GET_SLOT_POINTER(timer, expires) \
  (timer->slots + TIMER_GET_SLOT_INDEX(timer, expires))

```


## 相关代码
// fast_timer.h/fast_timer.c
// init the timer, slot_count=60 
int fast_timer_init(FastTimer *timer, const int slot_count,
    const int64_t current_time)
{
  int bytes;
  
  if (slot_count <= 0 || current_time <= 0) 
  {
    return EINVAL;
  }

  timer->slot_count = slot_count;  //default is 60 
  timer->base_time = current_time; //base time for slot 0
  timer->current_time = current_time;
  bytes = sizeof(FastTimerSlot) * slot_count;
  timer->slots = (FastTimerSlot *)malloc(bytes);
  
  if (timer->slots == NULL) 
  {
     return errno != 0 ? errno : ENOMEM;
  }
  
  memset(timer->slots, 0, bytes);
  return 0;
  
}

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
		pThreadData->ev_puller.iterator.count = ioevent_poll(&pThreadData->ev_puller); //若没有fd就绪，1秒后会超时返回（ioevent_init函数设置了timeout为1秒）
		
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
				&pThreadData->timer, g_current_time, &head); // get the expire event count，得到已经超时的事件
				
			if (count > 0)  // timeout event has been saved to head
			{
				deal_timeouts(&head); // process the timeout event，处理超时事件
			}
		}

        if (pThreadData->thread_loop_callback != NULL) 
		{
            pThreadData->thread_loop_callback(pThreadData);  // call this function every one loop 
        }
		
	}

	return 0;
}

// save expired FastTimerEntry to head
int fast_timer_timeouts_get(FastTimer *timer, const int64_t current_time,
   FastTimerEntry *head)
{
  FastTimerSlot *slot;
  FastTimerEntry *entry;
  FastTimerEntry *first;
  FastTimerEntry *last;
  FastTimerEntry *tail;
  int count;

  head->prev = NULL;
  head->next = NULL;
  
  if (timer->current_time >= current_time) // don't timeout, just return 
  {
    return 0;
  }

  first = NULL;
  last = NULL;
  tail = head;
  count = 0;
  
  while (timer->current_time < current_time) 
  {
    slot = TIMER_GET_SLOT_POINTER(timer, timer->current_time++); // 
    entry = slot->head.next;
	
    while (entry != NULL) 
	{
      if (entry->expires >= current_time) //not expired
	  {  
         if (first != NULL) 
		 {
            first->prev->next = entry;
            entry->prev = first->prev;

            tail->next = first;
            first->prev = tail;
            tail = last;
            first = NULL;
         }
		 
         if (entry->rehash) 
		 {
           last = entry;
           entry = entry->next;

           last->rehash = false;
           fast_timer_remove(timer, last);
           fast_timer_add(timer, last);
           continue;
         }
		 
      }
      else         //expired
	  {  
         count++;
		 
         if (first == NULL) 
		 {
            first = entry;
         }
      }

      last = entry;
      entry = entry->next;
    }

    if (first != NULL) 
	{
       first->prev->next = NULL;

       tail->next = first;
       first->prev = tail;
       tail = last;
       first = NULL;
    }
	
  }

  if (count > 0) 
  {
     tail->next = NULL;
  }

  return count;
}

//插入超时事件
// add entry to timer according to entry->expires
int fast_timer_add(FastTimer *timer, FastTimerEntry *entry)
{
  FastTimerSlot *slot;

  slot = TIMER_GET_SLOT_POINTER(timer, entry->expires >
     timer->current_time ? entry->expires : timer->current_time);
  entry->next = slot->head.next;
  
  if (slot->head.next != NULL) 
  {
    slot->head.next->prev = entry;
  }
  
  entry->prev = &slot->head;
  slot->head.next = entry;
  entry->rehash = false;
  return 0;
}


```
