---
title: fastdfs配置文件解析模块
date: 2016-11-06 19:11:20
categories: "fastdfs"
tags: [fastdfs]
---


### 加载配置文件解析
配置文件有： storage.conf,tracker.conf,mod_fastdfs.conf,http.conf等，配置文件中还可以用#include包含其他配置文件，
如#include http.conf。该配置文件解析模块就是读取这些配置文件，然后解析保存，方便得到其中的值。
如：
result=iniLoadFromFileEx(filename, &iniContext, true)
pBasePath = iniGetStrValue(NULL, "base_path", &iniContext); //该函数就可以得到配置文件中base_path的值
<!--more-->

相关参考：[http://slucx.blog.chinaunix.net/uid-29504236-id-4369694.html](http://slucx.blog.chinaunix.net/uid-29504236-id-4369694.html)

### 1. 相关数据结构
```
typedef struct
{
	IniSection global;  //保存全局key,value对
	HashArray sections;  //key is session name, and value is IniSection,这里hash数组用来保存[group1]下隶属于group1的<key,value>对
	IniSection *current_section; //for load from ini file，当前正在处理的IniSection
	char config_path[MAX_PATH_SIZE];  //save the config filepath, such as conf file is /etc/data/xxx.conf, config_path is /etc/data

    bool ignore_annotation; // 是否忽略注解，看代码时可以略过不看
} IniContext;


typedef struct
{
	IniItem *items;
	int count;  //item count
	int alloc_count;
} IniSection;

typedef struct
{
	char name[FAST_INI_ITEM_NAME_LEN + 1];
	char value[FAST_INI_ITEM_VALUE_LEN + 1];
} IniItem;


typedef struct tagHashArray
{
	HashData **buckets;
	HashFunc hash_func; // default is Time33Hash func
	int item_count;  // is all the item saved in buckets
	unsigned int *capacity; // pointer to prime global aarray, see hash_init_ex func
	double load_factor;
	int64_t max_bytes; // means the max size of space which can be used in hashArray
	int64_t bytes_used; // means the size of have been used, is the *capacity * sizeof(HashData *)，为已经使用的字节数
	bool is_malloc_capacity;
	// is_malloc_value is true: means the buffer of value is allocated in outer space, rather than beening allocated in the end of the key buffer
	bool is_malloc_value;  // it means hashArray whether  malloc space for value or not, default is false
	                       
	unsigned int lock_count;
	pthread_mutex_t *locks;
} HashArray;

typedef struct tagHashData
{
	int key_len;
	int value_len;
	int malloc_value_size;

#ifdef HASH_STORE_HASH_CODE
	unsigned int hash_code;
#endif

	char *value;
	struct tagHashData *next;
	char key[0];
} HashData;   // 元素

typedef int (*HashFunc) (const void *key, const int key_len);

static CDCPair g_dynamic_contents[_MAX_DYNAMIC_CONTENTS] = {{false, NULL, {0, 0, NULL}}}; //用一个全局数组来保存已经解析过的文件
typedef struct {
    bool used;
    IniContext *context;
    DynamicContents dynamicContents;
} CDCPair;
```

其中的sections是一个hash数组，用到的hash函数默认为Time33Hash函数
进行hash插入时，具体见hash_insert函数：
hash_code = pHash->hash_func(key, key_len);
ppBucket = pHash->buckets + (hash_code % (*pHash->capacity));

key是section_name, key_len是section_len，而
section_name, section_len为"[]"包围的字符串，见iniDoLoadItemsFromBuffer函数
如： [group1], section_name, section_len分别为group1和6
mod_fastdfs.conf文件中有：
```
#[group1]
#group_name=group1
#storage_server_port=23000
#store_path_count=2
#store_path0=/home/yuqing/fastdfs
#store_path1=/home/yuqing/fastdfs1
```
这里的group1就为section_name, section_len是6，此时current_section会指向一个新分配的IniSection，接着会将读取得到的
group_name等<key,value>对插入到current_section中去，然后插入到hash数组中去。
配置文件模块其实就是读取配置文件，然后初始化IniContext结构体。

### 2. 下面讲解各个函数
```
调用路径如下：
iniLoadFromFile -> iniLoadFromFileEx -> iniInitContext、iniDoLoadFromFile、iniSortItems、iniFreeContext
iniInitContext -> hash_init
hash_init->hash_init_ex->_hash_alloc_buckets
```

```
int iniLoadFromFile(const char *szFilename, IniContext *pContext)
{
    return iniLoadFromFileEx(szFilename, pContext, false);
}

int iniLoadFromFileEx(const char *szFilename, IniContext *pContext,
    bool ignore_annotation) // annotation is 注释, such as symbol = /* */
{
	int result;
	int len;
	char *pLast;
	char full_filename[MAX_PATH_SIZE]; //保存完整文件名路径

	if ((result=iniInitContext(pContext)) != 0)
	{
		return result;
	}

        pContext->ignore_annotation = ignore_annotation; //default is true

	if (strncasecmp(szFilename, "http://", 7) == 0) // szFilename可以为类似： http://www.abc.com/sdfs/xxx.conf
	{
		*pContext->config_path = '\0';
		snprintf(full_filename, sizeof(full_filename),"%s",szFilename); 
	}
	else
	{
		if (*szFilename == '/') // szFilename is the absolute path
		{
			pLast = strrchr(szFilename, '/'); //从后面开始查找第一个字符'/'
			len = pLast - szFilename;
			
			if (len >= sizeof(pContext->config_path))
			{
				logError("file: "__FILE__", line: %d, "\
					"the path of the config file: %s is " \
					"too long!", __LINE__, szFilename);
				return ENOSPC;
			}

			memcpy(pContext->config_path, szFilename, len);
			*(pContext->config_path + len) = '\0';
			snprintf(full_filename, sizeof(full_filename), \
				"%s", szFilename);
		}
		else  // 表明是从当前路径下的文件名，szFilename is the conf file name or is current_dir/xxx1/xxx2/xxx.conf
		{
			memset(pContext->config_path, 0, \
				sizeof(pContext->config_path));
			
			if (getcwd(pContext->config_path, sizeof( \
				pContext->config_path)) == NULL)  //getcwd函数为得到当前目录
			{
				logError("file: "__FILE__", line: %d, " \
					"getcwd fail, errno: %d, " \
					"error info: %s", \
					__LINE__, errno, STRERROR(errno));
				return errno != 0 ? errno : EPERM;
			}

			len = strlen(pContext->config_path);
			
			if (len > 0 && pContext->config_path[len - 1] == '/')
			{
				len--;
				*(pContext->config_path + len) = '\0';
			}  // make sure the last char is not '/'

			snprintf(full_filename, sizeof(full_filename), \
				"%s/%s", pContext->config_path, szFilename);

			pLast = strrchr(szFilename, '/');
			
			if (pLast != NULL)  // such as szFilename is "data/xxx.conf"
			{
				int tail_len;
				tail_len = pLast - szFilename;
				
				if (len + 1 + tail_len >= sizeof( \
						pContext->config_path))
				{
					logError("file: "__FILE__", line: %d, "\
						"the path of the config " \
						"file: %s is too long!", \
						__LINE__, szFilename);
					return ENOSPC;
				}

                *(pContext->config_path + len++) = '/';
				memcpy(pContext->config_path + len, \
					szFilename, tail_len);
				len += tail_len;
				*(pContext->config_path + len) = '\0';
			}
		}
	}

    // now full_filename is the absolute path of conf file, pContext->config_path
    // is the conf file directory
	result = iniDoLoadFromFile(full_filename, pContext);
	
	if (result == 0)
	{
		iniSortItems(pContext);
	}
	else
	{
		iniFreeContext(pContext);
	}

	return result;
}

// init the member of IniContext, such as init the hash member:pContext->sections
static int iniInitContext(IniContext *pContext)
{
	int result;

	memset(pContext, 0, sizeof(IniContext));
	pContext->current_section = &pContext->global; //指向global,用于保存全局key,value对
	
	if ((result=hash_init(&pContext->sections, Time33Hash, 32, 0.75)) != 0) //hash数组初始化，Time33Hash为hash函数，32为容量，0.75为负载因子
	{
		logError("file: "__FILE__", line: %d, " \
			"hash_init fail, errno: %d, error info: %s", \
			__LINE__, result, STRERROR(result));
	}

	return result;
}

hash函数如下：
#define TIME33_HASH_FUNC(init_value) \
	int nHash; \
	unsigned char *pKey; \
	unsigned char *pEnd; \
 \
	nHash = init_value; \
	pEnd = (unsigned char *)key + key_len; \
	for (pKey = (unsigned char *)key; pKey != pEnd; pKey++) \
	{ \
		nHash += (nHash << 5) + (*pKey); \
	} \
 \
	return nHash; \


// get a int value according to the key value
int Time33Hash(const void *key, const int key_len)
{
	TIME33_HASH_FUNC(0)
}

#define hash_init(pHash, hash_func, capacity, load_factor) \
	hash_init_ex(pHash, hash_func, capacity, load_factor, 0, false)

// 参数依次为：要初始化的hash数组，hash函数，容量，负载因子（当前保存的项数/capacity）,hash数组能用的最大字节数，保存value的空间是否已经在外部被分配了
int hash_init_ex(HashArray *pHash, HashFunc hash_func, \
		const unsigned int capacity, const double load_factor, \
		const int64_t max_bytes, const bool bMallocValue)
{
	unsigned int *pprime;
	unsigned int *prime_end;
	int result;

	memset(pHash, 0, sizeof(HashArray));
	prime_end = prime_array + PRIME_ARRAY_SIZE;  // 素数数组
	
	for (pprime = prime_array; pprime!=prime_end; pprime++)
	{
		if ( *pprime > capacity ) //找到第一个大于容量的素数
		{
			pHash->capacity = pprime;
			break;
		}
	}

	if (pHash->capacity == NULL)
	{
		return EINVAL;
	}

	if ((result=_hash_alloc_buckets(pHash, 0)) != 0) //分配桶
	{
		return result;
	}

	pHash->hash_func = hash_func;
	pHash->max_bytes = max_bytes; //hash数组能够使用的最大字节数
	pHash->is_malloc_value = bMallocValue;  // default is false，保存key,value中的value数据时的空间是否已经在外部被分配

    // load_factor default is 0.75
	if (load_factor >= 0.00 && load_factor <= 1.00)
	{
		pHash->load_factor = load_factor;
	}
	else
	{
		pHash->load_factor = 0.50;
	}

	return 0;
}

// 素数数组
static unsigned int prime_array[] = {
    1,              /* 0 */
    3,              /* 1 */
    17,             /* 2 */
    37,             /* 3 */
    79,             /* 4 */
    163,            /* 5 */
    331,            /* 6 */
    673,            /* 7 */
    1361,           /* 8 */
    2729,           /* 9 */
    5471,           /* 10 */
    10949,          /* 11 */
    21911,          /* 12 */
    43853,          /* 13 */
    87719,          /* 14 */
    175447,         /* 15 */
    350899,         /* 16 */
    701819,         /* 17 */
    1403641,        /* 18 */
    2807303,        /* 19 */
    5614657,        /* 20 */
    11229331,       /* 21 */
    22458671,       /* 22 */
    44917381,       /* 23 */
    89834777,       /* 24 */
    179669557,      /* 25 */
    359339171,      /* 26 */
    718678369,      /* 27 */
    1437356741,     /* 28 */
    2147483647      /* 29 (largest signed int prime) */
};

#define PRIME_ARRAY_SIZE  30

// allocate the space of hash array
static int _hash_alloc_buckets(HashArray *pHash, const unsigned int old_capacity)
{
	size_t bytes;

	bytes = sizeof(HashData *) * (*pHash->capacity);
	
	if (pHash->max_bytes > 0 && pHash->bytes_used+bytes > pHash->max_bytes)
	{
		return ENOSPC; // no more memory in device 
	}

	pHash->buckets = (HashData **)malloc(bytes);
	
	if (pHash->buckets == NULL)
	{
		return ENOMEM;
	}

	memset(pHash->buckets, 0, bytes); // sizeof(HashData *) * old_capacity为旧数组的大小
	pHash->bytes_used += bytes - sizeof(HashData *) * old_capacity; //bytes为新大小，减去旧数组大小则为新增大小

	return 0;
}



static int iniDoLoadFromFile(const char *szFilename, \
		IniContext *pContext)
{
	char *content;
	int result;
	int http_status;
	int content_len;
	int64_t file_size;
	char error_info[512];

	if (strncasecmp(szFilename, "http://", 7) == 0) // 是否为http://xxx.xx.xx/xx1/xx.conf形式
	{
	    // szFilename: http://xx1.xx2.xx3/haha/dir/xxx.conf
		if ((result=get_url_content(szFilename, 10, 60, &http_status, \   
				&content, &content_len, error_info)) != 0)  //get_url_content函数发送http请求获得文件内容
		{
			logError("file: "__FILE__", line: %d, " \
				"get_url_content fail, " \
				"url: %s, error info: %s", \
				__LINE__, szFilename, error_info);
			return result;
		}

		if (http_status != 200) // means http response status is not correct
		{
			free(content);
			logError("file: "__FILE__", line: %d, " \
				"HTTP status code: %d != 200, url: %s", \
				__LINE__, http_status, szFilename);
			return EINVAL;
		}
		
	}
	else
	{
		if ((result=getFileContent(szFilename, &content, \
				&file_size)) != 0)
		{
			return result;
		}
	}

	result = iniLoadItemsFromBuffer(content, pContext);
	free(content);

	return result;
}


int get_url_content(const char *url, const int connect_timeout, \
	const int network_timeout, int *http_status, \
	char **content, int *content_len, char *error_info)
{
    *content = NULL;
    return get_url_content_ex(url, strlen(url), connect_timeout, network_timeout,
            http_status, content, content_len, error_info);
}

/*
get the content of url, the func will send http request and recv http response
// connect_timeout is 10 default
// network_timeout is 60 default
http_status用于保存http响应报文的状态，如http 1.1 200 ok中的200
content和content_len用于保存内容和长度
error_info用于保存错误信息
*/
int get_url_content_ex(const char *url, const int url_len,
        const int connect_timeout, const int network_timeout,
        int *http_status, char **content, int *content_len, char *error_info)
{
	char domain_name[256];
	char ip_addr[IP_ADDRESS_SIZE];
	char out_buff[4096];
	int domain_len;
	int out_len;
	int alloc_size;
	int recv_bytes;
	int result;
	int sock;
	int port;
    bool bNeedAlloc;
	const char *pDomain;
	const char *pContent;
	const char *pURI;
	char *pPort;
	char *pSpace;

	*http_status = 0;
	
    if (*content == NULL)
    {
        bNeedAlloc = true;
        alloc_size = 64 * 1024;
    }
    else
    {
        bNeedAlloc = false;
        alloc_size = *content_len - 1;
    }
	
	*content_len = 0;
	
    if (url_len > sizeof(out_buff) - 128)
    {
		sprintf(error_info, "file: "__FILE__", line: %d, "
                "url too long, url length: %d > %d", __LINE__,
                url_len, (int)(sizeof(out_buff) - 128));

		return ENAMETOOLONG;
    }

	if (url_len <= 7 || strncasecmp(url, "http://", 7) != 0)
	{
		sprintf(error_info, "file: "__FILE__", line: %d, " \
			"invalid url.", __LINE__);
		return EINVAL;
	}

	pDomain = url + 7;
	pURI = strchr(pDomain, '/');
	
	if (pURI == NULL)
	{
		domain_len = url_len - 7;
		pURI = "/";
	}
	else
	{
		domain_len = pURI - pDomain;
	}

	if (domain_len >= sizeof(domain_name))
	{
		sprintf(error_info, "file: "__FILE__", line: %d, " \
			"domain is too large, exceed %d.", \
			__LINE__, (int)sizeof(domain_name));
		return EINVAL;
	}

	memcpy(domain_name, pDomain, domain_len);
	*(domain_name + domain_len) = '\0';
	pPort = strchr(domain_name, ':');
	
	if (pPort == NULL)
	{
		port = 80;
	}
	else
	{
		*pPort = '\0';
		port = atoi(pPort + 1);
	}

	if (getIpaddrByName(domain_name, ip_addr, \
		sizeof(ip_addr)) == INADDR_NONE)
	{
		sprintf(error_info, "file: "__FILE__", line: %d, " \
			"resolve domain \"%s\" fail.", \
			__LINE__, domain_name);
		return EINVAL;
	}

	sock = socket(AF_INET, SOCK_STREAM, 0);
	
	if(sock < 0)
	{
		sprintf(error_info, "file: "__FILE__", line: %d, " \
			"socket create failed, errno: %d, " \
			"error info: %s", __LINE__, \
			errno, STRERROR(errno));
		return errno != 0 ? errno : EPERM;
	}

	if ((result=connectserverbyip_nb_auto(sock, ip_addr, port, \
			connect_timeout)) != 0)
	{
		close(sock);

		sprintf(error_info, "file: "__FILE__", line: %d, " \
			"connect to %s:%d fail, errno: %d, " \
			"error info: %s", __LINE__, domain_name, \
			port, result, STRERROR(result));

		return result;
	}

	out_len = snprintf(out_buff, sizeof(out_buff), \
		"GET %s HTTP/1.0\r\n" \
		"Host: %s:%d\r\n" \
		"Connection: close\r\n" \
		"\r\n", pURI, domain_name, port);
	// we have Connection: close means: the peer will shutdown the socket when it has finished sending data
	// send http request 
	if ((result=tcpsenddata(sock, out_buff, out_len, network_timeout)) != 0)
	{
		close(sock);

		sprintf(error_info, "file: "__FILE__", line: %d, " \
			"send data to %s:%d fail, errno: %d, " \
			"error info: %s", __LINE__, domain_name, \
			port, result, STRERROR(result));

		return result;
	}

    if (bNeedAlloc)
    {
        *content = (char *)malloc(alloc_size + 1);
		
        if (*content == NULL)
        {
            close(sock);
            result = errno != 0 ? errno : ENOMEM;

            sprintf(error_info, "file: "__FILE__", line: %d, " \
                    "malloc %d bytes fail, errno: %d, " \
                    "error info: %s", __LINE__, alloc_size + 1, \
                    result, STRERROR(result));

            return result;
        }
    }

	do
	{
		recv_bytes = alloc_size - *content_len; // recv_bytes: left space to recv data
		
		if (recv_bytes <= 0)
		{
            if (bNeedAlloc)
            {
                alloc_size *= 2;
                *content = (char *)realloc(*content, alloc_size + 1);
				
                if (*content == NULL)
                {
                    *content_len = 0;
                    close(sock);
                    result = errno != 0 ? errno : ENOMEM;

                    sprintf(error_info, "file: "__FILE__", line: %d, " \
                            "realloc %d bytes fail, errno: %d, " \
                            "error info: %s", __LINE__, \
                            alloc_size + 1, \
                            result, STRERROR(result));

                    return result;
                }

                recv_bytes = alloc_size - *content_len;
            }
            else
            {
                    sprintf(error_info, "file: "__FILE__", line: %d, " \
                            "buffer size: %d is too small", \
                            __LINE__, alloc_size);
                    return ENOSPC;
            }
			
		}

		result = tcprecvdata_ex(sock, *content + *content_len, \
				recv_bytes, network_timeout, &recv_bytes);

		*content_len += recv_bytes;
	} while (result == 0);

    do
    {
        if (result == ENOTCONN) // means the peer has shutdowned the socket
        {
            result = 0;   // success value
        }
        else 
	{
            sprintf(error_info, "file: "__FILE__", line: %d, " \
                    "recv data from %s:%d fail, errno: %d, " \
                    "error info: %s", __LINE__, domain_name, \
                    port, result, STRERROR(result));

            break;
        }

        *(*content + *content_len) = '\0';
        pContent = strstr(*content, "\r\n\r\n");
		
        if (pContent == NULL)
        {
            sprintf(error_info, "file: "__FILE__", line: %d, " \
                    "response data from %s:%d is invalid", \
                    __LINE__, domain_name, port);

            result = EINVAL;
            break;
        }

        pContent += 4;   // pointer to the next line
        pSpace = strchr(*content, ' ');
		
        if (pSpace == NULL || pSpace >= pContent)
        {
            sprintf(error_info, "file: "__FILE__", line: %d, " \
                    "response data from %s:%d is invalid", \
                    __LINE__, domain_name, port);

            result = EINVAL;
            break;
        }

        *http_status = atoi(pSpace + 1);  // http response status: such as http/1.1 200 ok
        *content_len -= pContent - *content;  // minus the length of status line: http/1.1 200 ok
        memcpy(*content, pContent, *content_len);  // remove the status line
        *(*content + *content_len) = '\0';
        *error_info = '\0';
    } while (0);

	close(sock);
	
    if (result != 0 && bNeedAlloc)  // result !=0: means error occured
    {
        free(*content);
        *content = NULL;
        *content_len = 0;
    }

	return result;
}

// return the string ip addr by ip name(maybe is digital ip addr or domain addr)
in_addr_t getIpaddrByName(const char *name, char *buff, const int bufferSize)
{
	struct in_addr ip_addr;
	struct hostent *ent;
	in_addr_t **addr_list;

	if ((*name >= '0' && *name <= '9') &&   // name is the digital ip address 
		inet_pton(AF_INET, name, &ip_addr) == 1)  // success
	{
		if (buff != NULL)
		{
			snprintf(buff, bufferSize, "%s", name);
		}
		
		return ip_addr.s_addr;
	}

	ent = gethostbyname(name);
	
	if (ent == NULL)
	{
		return INADDR_NONE;
	}

    addr_list = (in_addr_t **)ent->h_addr_list;
	
	if (addr_list[0] == NULL)
	{
		return INADDR_NONE;
	}

	memset(&ip_addr, 0, sizeof(ip_addr));
	ip_addr.s_addr = *(addr_list[0]);
	
	if (buff != NULL)
	{
		if (inet_ntop(AF_INET, &ip_addr, buff, bufferSize) == NULL)
		{
			*buff = '\0';
		}
	}

	return ip_addr.s_addr;
}


/** connect to server by non-block mode, auto detect socket block mode
 *  parameters:
 *          sock: the socket, can be block mode
 *          server_ip: ip address of the server
 *          server_port: port of the server
 *          timeout: connect timeout in seconds
 *  return: error no, 0 success, != 0 fail
*/
#define connectserverbyip_nb_auto(sock, server_ip, server_port, timeout) \
	connectserverbyip_nb_ex(sock, server_ip, server_port, timeout, true)

// connect the server by ip , return 0 stands for connect succeed
int connectserverbyip_nb_ex(int sock, const char *server_ip, \
		const short server_port, const int timeout, \
		const bool auto_detect)
{
	int result;
	int flags;
	bool needRestore;
	socklen_t len;

#ifdef USE_SELECT
	fd_set rset;
	fd_set wset;
	struct timeval tval;
#else
	struct pollfd pollfds;
#endif

	struct sockaddr_in addr;
	struct sockaddr_in6 addr6;
    void *dest;
    int size;

    memset(&addr, 0, sizeof(struct sockaddr_in));
    memset(&addr6, 0, sizeof(struct sockaddr_in6));

    if ((result=setsockaddrbyip(server_ip, server_port, &addr, &addr6,
                    &dest, &size)) != 0)
    {
        return result;
    }

	if (auto_detect)
	{
		flags = fcntl(sock, F_GETFL, 0);
		
		if (flags < 0)
		{
			return errno != 0 ? errno : EACCES;
		}

		if ((flags & O_NONBLOCK) == 0)
		{
			if (fcntl(sock, F_SETFL, flags | O_NONBLOCK) < 0)
			{
				return errno != 0 ? errno : EACCES;
			}

			needRestore = true;
		}
		else
		{
			needRestore = false;
		}
	}
	else
	{
		needRestore = false;
		flags = 0;
	}

	do
	{
		if (connect(sock, (const struct sockaddr*)dest, size) < 0)
		{
			result = errno != 0 ? errno : EINPROGRESS;
			
			if (result != EINPROGRESS)
			{
				break;
			}
		}
		else  // connect return 0: success, -1: failure
		{
			result = 0;
			break;
		}

        // after call connect, we call select or poll to get error if error occurs

#ifdef USE_SELECT
		FD_ZERO(&rset);
		FD_ZERO(&wset);
		FD_SET(sock, &rset);
		FD_SET(sock, &wset);
		tval.tv_sec = timeout;
		tval.tv_usec = 0;
		
		result = select(sock+1, &rset, &wset, NULL, \
				timeout > 0 ? &tval : NULL);
#else
		pollfds.fd = sock;
		pollfds.events = POLLIN | POLLOUT;
		result = poll(&pollfds, 1, 1000 * timeout);
#endif

		if (result == 0)
		{
			result = ETIMEDOUT;
			break;
		}
		else if (result < 0)
		{
			result = errno != 0 ? errno : EINTR;
			break;
		}

		// means return value > 0
		len = sizeof(result);

		// getsockopt: 0: success, -1:failure
		if (getsockopt(sock, SOL_SOCKET, SO_ERROR, &result, &len) < 0)
		{
			result = errno != 0 ? errno : EACCES; // means failure
			break;
		}
		
		
	} while (0);

	if (needRestore)
	{
		fcntl(sock, F_SETFL, flags);
	}
  
	return result;
}



/*
get the content of filename, which is saved to *buff, the size of filename is saved to file_size
*/
int getFileContent(const char *filename, char **buff, int64_t *file_size)
{
	int fd;

    if (!isFile(filename))
    {
		*buff = NULL;
		*file_size = 0;
		logError("file: "__FILE__", line: %d, "
                "%s is not a regular file", __LINE__, filename);
        return EINVAL;
    }

	fd = open(filename, O_RDONLY);
	
	if (fd < 0)
	{
		*buff = NULL;
		*file_size = 0;
		logError("file: "__FILE__", line: %d, " \
			"open file %s fail, " \
			"errno: %d, error info: %s", __LINE__, \
			filename, errno, STRERROR(errno));
		return errno != 0 ? errno : ENOENT;
	}

	if ((*file_size=lseek(fd, 0, SEEK_END)) < 0)
	{
		*buff = NULL;
		*file_size = 0;
		close(fd);
		logError("file: "__FILE__", line: %d, " \
			"lseek file %s fail, " \
			"errno: %d, error info: %s", __LINE__, \
			filename, errno, STRERROR(errno));
		return errno != 0 ? errno : EIO;
	}

	*buff = (char *)malloc(*file_size + 1);
	
	if (*buff == NULL)
	{
		*file_size = 0;
		close(fd);

		logError("file: "__FILE__", line: %d, " \
			"malloc %d bytes fail", __LINE__, \
			(int)(*file_size + 1));
		return errno != 0 ? errno : ENOMEM;
	}

	if (lseek(fd, 0, SEEK_SET) < 0)
	{
		*buff = NULL;
		*file_size = 0;
		close(fd);
		logError("file: "__FILE__", line: %d, " \
			"lseek file %s fail, " \
			"errno: %d, error info: %s", __LINE__, \
			filename, errno, STRERROR(errno));
		return errno != 0 ? errno : EIO;
	}
	
	if (read(fd, *buff, *file_size) != *file_size)
	{
		free(*buff);
		*buff = NULL;
		*file_size = 0;
		close(fd);
		logError("file: "__FILE__", line: %d, " \
			"read from file %s fail, " \
			"errno: %d, error info: %s", __LINE__, \
			filename, errno, STRERROR(errno));
		return errno != 0 ? errno : EIO;
	}

	(*buff)[*file_size] = '\0';
	close(fd);

	return 0;
}

static int iniLoadItemsFromBuffer(char *content, IniContext *pContext)
{
    char *pContent;
    char *new_content;
    int content_len;
    int new_content_len;

    new_content = content;
    new_content_len = strlen(content);

    do
    {
        pContent = new_content; // after get rid of first #@if, process left #@if
        content_len = new_content_len;
		
        if ((new_content=iniProccessIf(pContent, content_len,
                        pContext, &new_content_len)) == NULL)
        {
            return ENOMEM;
        }

		
    } while (new_content != pContent);

    do
    {
        pContent = new_content;
        content_len = new_content_len;
		
        if ((new_content=iniProccessFor(pContent, content_len,
                        pContext, &new_content_len)) == NULL)
        {
            return ENOMEM;
		
        }
    } while (new_content != pContent);  // loop processing

    return iniDoLoadItemsFromBuffer(new_content, pContext);
	
}

/*
function: reslove the content, and get rid of #@if and #@endif, save to new buffer
returnd by iniProcessIf, such as:
content is:
#@if xxx
...abc
#@endif
...def

new buffer is: 
...abc
...def
returned by iniProcessIf

该函数看不懂感觉可以略过不看
*/
static char *iniProccessIf(char *content, const int content_len,
        IniContext *pContext, int *new_content_len)
{
    char *pStart;
    char *pEnd;
    char *pCondition;
    char *pElse;
    char *pIfPart;
    char *pElsePart;
    int conditionLen;
    int ifPartLen;
    int elsePartLen;
    int copyLen;
    char *newContent;
    char *pDest;

    *new_content_len = content_len;
	
    pStart = strstr(content, _PREPROCESS_TAG_STR_IF);
	
    if (pStart == NULL)
    {
        return content;
    }
	
    pCondition = pStart + _PREPROCESS_TAG_LEN_IF;
    pIfPart = strchr(pCondition, '\n');
	
    if (pIfPart == NULL)
    {
        return content;
    }
	
    conditionLen = pIfPart - pCondition;

    pEnd = strstr(pIfPart, _PREPROCESS_TAG_STR_ENDIF);
	
    if (pEnd == NULL)
    {
        return content;
    }

    pElse = strstr(pIfPart, _PREPROCESS_TAG_STR_ELSE);
	
    if (pElse == NULL || pElse > pEnd)
    {
        ifPartLen = pEnd - pIfPart;
        pElsePart = NULL;
        elsePartLen = 0;
    }
    else
    {
        ifPartLen = pElse - pIfPart;
        pElsePart = strchr(pElse + _PREPROCESS_TAG_LEN_ELSE, '\n');
		
        if (pElsePart == NULL)
        {
            return content;
        }

        elsePartLen = pEnd - pElsePart;
    }

    newContent = iniAllocContent(pContext, content_len); // newContent is the buffer address
	
    if (newContent == NULL)
    {
        return NULL;
    }

    pDest = newContent;
    copyLen = pStart - content;
	
    if (copyLen > 0)
    {
        memcpy(pDest, content, copyLen);
        pDest += copyLen;
    }

    if (iniCalcCondition(pCondition, conditionLen))
    {
        if (ifPartLen > 0)
        {
            memcpy(pDest, pIfPart, ifPartLen);
            pDest += ifPartLen;
        }
    }
    else
    {
        if (elsePartLen > 0)
        {
            memcpy(pDest, pElsePart, elsePartLen);
            pDest += elsePartLen;
        }
    }

    copyLen = (content + content_len) - (pEnd + _PREPROCESS_TAG_LEN_ENDIF);
	
    if (copyLen > 0)
    {
        memcpy(pDest, pEnd + _PREPROCESS_TAG_LEN_ENDIF, copyLen);
        pDest += copyLen;
    }

    *pDest = '\0';   
    *new_content_len = pDest - newContent;
    return newContent;
}

/* process the for block: #@for  ... #@endfor
get rid of the #@endfor, #@for, and expand the for loop which replace {$i} with the real value
and saved to new buffer returned by iniProcessFor
该函数与上一个函数类似

*/
static char *iniProccessFor(char *content, const int content_len,
        IniContext *pContext, int *new_content_len)
{
    char *pStart;
    char *pEnd;
    char *pForRange;
    char *pForBlock;
    char *id;
    char tag[80];
    char value[16];
    int idLen;
    int rangeLen;
    int forBlockLen;
    int start;
    int end;
    int step;
    int count;
    int i;
    int copyLen;
    int tagLen;
    int valueLen;
    char *newContent;
    char *pDest;

    *new_content_len = content_len;
    pStart = strstr(content, _PREPROCESS_TAG_STR_FOR);
	
    if (pStart == NULL)
    {
        return content;
    }
	
    pForRange = pStart + _PREPROCESS_TAG_LEN_FOR; // for condition begin
    pForBlock = strchr(pForRange, '\n');  // for block begin
	
    if (pForBlock == NULL)
    {
        return content;
    }
	
    rangeLen = pForBlock - pForRange;

    pEnd = strstr(pForBlock, _PREPROCESS_TAG_STR_ENDFOR);
	
    if (pEnd == NULL)
    {
        return content;
    }
	
    forBlockLen = pEnd - pForBlock;  // for block len

    if (iniParseForRange(pForRange, rangeLen, &id, &idLen,
                &start, &end, &step) != 0)
    {
        return NULL;
    }
	
    if (step == 0)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid step: %d for range: %.*s", __LINE__,
                step, rangeLen, pForRange);
        return NULL;
    }
	
    count = (end - start) / step;  // how many steps
	
    if (count < 0)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid step: %d for range: %.*s", __LINE__,
                step, rangeLen, pForRange);
        return NULL;
    }

    newContent = iniAllocContent(pContext, content_len + (forBlockLen + 16) * count);
	
    if (newContent == NULL)
    {
        return NULL;
    }

    pDest = newContent;  // newContent is the buffer addr to stored for block content
    copyLen = pStart - content;
	
    if (copyLen > 0)
    {
        memcpy(pDest, content, copyLen);
        pDest += copyLen;
    }

	// id="i", idLen=1, tag will be "{$i}", tagLen will be 4 = strlen("{$i}")
    tagLen = sprintf(tag, "{$%.*s}", idLen, id);
	// tagLen is the variable length
	
    for (i=start; i<=end; i+=step)
    {
        char *p;
        char *pRemain;
        int remainLen;

        valueLen = sprintf(value, "%d", i);

        pRemain = pForBlock;
        remainLen = forBlockLen;
		
        while (remainLen >= tagLen)
        {
            p = (char *)memmem(pRemain, remainLen, tag, tagLen);
			
            if (p == NULL)
            {
                memcpy(pDest, pRemain, remainLen);
                pDest += remainLen;
                break;
            }

            copyLen = p - pRemain;
			
            if (copyLen > 0)
            {
                memcpy(pDest, pRemain, copyLen);
                pDest += copyLen;
            }
			
            memcpy(pDest, value, valueLen);
            pDest += valueLen;

            pRemain = p + tagLen;
            remainLen -= copyLen + tagLen;
        }
    }

    copyLen = (content + content_len) - (pEnd + _PREPROCESS_TAG_LEN_ENDFOR);
	
    if (copyLen > 0)
    {
        memcpy(pDest, pEnd + _PREPROCESS_TAG_LEN_ENDFOR, copyLen);
        pDest += copyLen;
    }

    *pDest = '\0';
    *new_content_len = pDest - newContent;
    return newContent;
}

//alloc space for the length of content_len, return the buffer addr
static char *iniAllocContent(IniContext *pContext, const int content_len)
{
    char *buff;
    DynamicContents *pDynamicContents;
    pDynamicContents = iniAllocDynamicContent(pContext);
	
    if (pDynamicContents == NULL)
    {
        logError("file: "__FILE__", line: %d, "
                "malloc dynamic contents fail", __LINE__);
        return NULL;
    }

	// default: pDynamicContents->count = 0, pDynamicContents->alloc_count=0
    if (pDynamicContents->count >= pDynamicContents->alloc_count)
    {
        int alloc_count;
        int bytes;
        char **contents;
		
        if (pDynamicContents->alloc_count == 0)
        {
            alloc_count = 8;
        }
        else
        {
            alloc_count = pDynamicContents->alloc_count * 2;
        }
		
        bytes = sizeof(char *) * alloc_count;
        contents = (char **)malloc(bytes);
		
        if (contents == NULL)
        {
            logError("file: "__FILE__", line: %d, "
                    "malloc %d bytes fail", __LINE__, bytes);
            return NULL;
        }
		
        memset(contents, 0, bytes);
		
        if (pDynamicContents->count > 0)
        {
            memcpy(contents, pDynamicContents->contents,
                    sizeof(char *) * pDynamicContents->count);
            free(pDynamicContents->contents);
        }
		
        pDynamicContents->contents = contents;
        pDynamicContents->alloc_count = alloc_count;
    }

    buff = malloc(content_len);
	
    if (buff == NULL)
    {
        logError("file: "__FILE__", line: %d, "
                "malloc %d bytes fail", __LINE__, content_len);
        return NULL;
    }
	
    pDynamicContents->contents[pDynamicContents->count++] = buff;
    return buff;
}

// find pContext in g_dynamic_contents, if find: return, or add pContext to g_dynamic_contents
static DynamicContents *iniAllocDynamicContent(IniContext *pContext)
{
    int i;
	
    if (g_dynamic_contents[g_dynamic_content_index].context == pContext)
    {
        return &g_dynamic_contents[g_dynamic_content_index].dynamicContents;
    }

    if (g_dynamic_content_count > 0)
    {
        for (i=0; i<_MAX_DYNAMIC_CONTENTS; i++)
        {
            if (g_dynamic_contents[i].context == pContext)
            {
                g_dynamic_content_index = i;
                return &g_dynamic_contents[g_dynamic_content_index].dynamicContents;
            }
        }
    }

    if (g_dynamic_content_count == _MAX_DYNAMIC_CONTENTS)
    {
        return NULL;
    }

    for (i=0; i<_MAX_DYNAMIC_CONTENTS; i++)
    {
        if (!g_dynamic_contents[i].used)
        {
            g_dynamic_contents[i].used = true;
            g_dynamic_contents[i].context = pContext;
            g_dynamic_content_index = i;
            g_dynamic_content_count++;
            return &g_dynamic_contents[g_dynamic_content_index].dynamicContents;
        }
    }

    return NULL;
}



/*
function: resolve the format:
%{LOCAL_IP} in [10.0.11.89,10.0.11.99]
%{LOCAL_HOST} in [10.0.11.89,10.0.11.99]
and compare g_local_host_ip_addrs array, check every elem in g_local_host_ip_addrs is
whether exists in [10.0.11.89,10.0.11.99] or not, exists return true or return false
*/
static bool iniCalcCondition(char *condition, const int condition_len)
{
    /*
     * current only support %{VARIABLE} in [x,y,..]
     * support variables are: LOCAL_IP and LOCAL_HOST
     * such as: %{LOCAL_IP} in [10.0.11.89,10.0.11.99]
     **/
#define _PREPROCESS_VARIABLE_TYPE_LOCAL_IP   1
#define _PREPROCESS_VARIABLE_TYPE_LOCAL_HOST 2
#define _PREPROCESS_MAX_LIST_VALUE_COUNT    32
    char *p;
    char *pEnd;
    char *pSquareEnd;
    char *values[_PREPROCESS_MAX_LIST_VALUE_COUNT];
    int varType;
    int count;
    int i;

    pEnd = condition + condition_len;
    p = pEnd - 1;
	
    while (p > condition && (*p == ' ' || *p == '\t'))
    {
        p--;
    }
	
    if (*p != ']')
    {
		logWarning("file: "__FILE__", line: %d, "
                "expect \"]\", condition: %.*s", __LINE__,
                condition_len, condition);
        return false;
    }
	
    pSquareEnd = p;

    p = condition;
	
    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }

    if (pEnd - p < 12)
    {
		logWarning("file: "__FILE__", line: %d, "
                "unkown condition: %.*s", __LINE__,
                condition_len, condition);
        return false;
    }

    if (memcmp(p, _PREPROCESS_VARIABLE_STR_LOCAL_IP,
                _PREPROCESS_VARIABLE_LEN_LOCAL_IP) == 0)
    {
        varType = _PREPROCESS_VARIABLE_TYPE_LOCAL_IP;
        p += _PREPROCESS_VARIABLE_LEN_LOCAL_IP;
    }
    else if (memcmp(p, _PREPROCESS_VARIABLE_STR_LOCAL_HOST,
                _PREPROCESS_VARIABLE_LEN_LOCAL_HOST) == 0)
    {
        varType = _PREPROCESS_VARIABLE_TYPE_LOCAL_HOST;
        p += _PREPROCESS_VARIABLE_LEN_LOCAL_HOST;
    }
    else
    {
		logWarning("file: "__FILE__", line: %d, "
                "unkown condition: %.*s", __LINE__,
                condition_len, condition);
        return false;
    }

    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }
	
    if (pEnd - p < 4 || memcmp(p, "in", 2) != 0)
    {
		logWarning("file: "__FILE__", line: %d, "
                "expect \"in\", condition: %.*s", __LINE__,
                condition_len, condition);
        return false;
    }
	
    p += 2;  //skip in

    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }
	
    if (*p != '[')
    {
		logWarning("file: "__FILE__", line: %d, "
                "expect \"[\", condition: %.*s", __LINE__,
                condition_len, condition);
        return false;
    }

    *pSquareEnd = '\0';
    count = splitEx(p + 1, ',', values, _PREPROCESS_MAX_LIST_VALUE_COUNT);
	
    for (i=0; i<count; i++)
    {
        values[i] = trim(values[i]);
    }
	
    if (varType == _PREPROCESS_VARIABLE_TYPE_LOCAL_HOST)
    {
        char host[128];
		
        if (gethostname(host, sizeof(host)) != 0)
        {
            logWarning("file: "__FILE__", line: %d, "
                    "call gethostname fail, "
                    "errno: %d, error info: %s", __LINE__,
                    errno, STRERROR(errno));
            return false;
        }
		
        return iniMatchValue(host, values, count);
    }
    else
    {
        const char *local_ip;
        local_ip = get_first_local_ip();
		
        while (local_ip != NULL)
        {
            if (iniMatchValue(local_ip, values, count))
            {
                return true;
            }
			
            local_ip = get_next_local_ip(local_ip);
        }
    }

    return false;
}


// find target in array: values, if find return true, else return false
static bool iniMatchValue(const char *target, char **values, const int count)
{
    int i;
    for (i=0; i<count; i++)
    {
        if (strcmp(target, values[i]) == 0)
        {
            return true;
        }
    }

    return false;
}

// get a integer from str ended by pEnd, nlen is the length of integer
static char *iniGetInteger(char *str, char *pEnd, int *nlen)
{
    char *p;
    char *pNumber;

    p = str;
	
    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }

    pNumber = p;
	
    while (p < pEnd && (*p >= '0' && *p <= '9'))
    {
        p++;
    }

    *nlen = p - pNumber;
    return pNumber;
}

// parse the format: #@for i from 0 to 15 step 1
// *id pointer to i, idLen=1, start=0, end=15, step=1
static int iniParseForRange(char *range, const int range_len,
        char **id, int *idLen, int *start, int *end, int *step)
{
    /**
     *
     * #@for i from 0 to 15 step 1
     */


    char *p;
    char *pEnd;
    char *pNumber;
    int nlen;

    pEnd = range + range_len;
    p = range;
	
    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }

    if (pEnd - p < 10)
    {
		logWarning("file: "__FILE__", line: %d, "
                "unkown for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }

    *id = p;
	
    while (p < pEnd && !(*p == ' ' || *p == '\t'))
    {
        p++;
    }
	
    *idLen = p - *id;
	
    if (*idLen == 0 || *idLen > 64)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }

    if (pEnd - p < 8)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }

    p++;
	
    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }
	
    if (!(memcmp(p, _PREPROCESS_TAG_STR_FOR_FROM,
                    _PREPROCESS_TAG_LEN_FOR_FROM) == 0 &&
                (*(p+_PREPROCESS_TAG_LEN_FOR_FROM) == ' ' ||
                 *(p+_PREPROCESS_TAG_LEN_FOR_FROM) == '\t')))
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    p += _PREPROCESS_TAG_LEN_FOR_FROM + 1;
    pNumber = iniGetInteger(p, pEnd, &nlen);
	
    if (nlen == 0)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    *start = atoi(pNumber);  //atoi meets non-numbers will stop transfer
    p = pNumber + nlen;

    if (pEnd - p < 4 || !(*p == ' ' || *p == '\t'))
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    p++;
	
    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }
	
    if (!(memcmp(p, _PREPROCESS_TAG_STR_FOR_TO,
                    _PREPROCESS_TAG_LEN_FOR_TO) == 0 &&
                (*(p+_PREPROCESS_TAG_LEN_FOR_TO) == ' ' ||
                 *(p+_PREPROCESS_TAG_LEN_FOR_TO) == '\t')))
    {
		logWarning("file: "__FILE__", line: %d, "
                "unkown for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    p += _PREPROCESS_TAG_LEN_FOR_TO + 1;
    pNumber = iniGetInteger(p, pEnd, &nlen);
	
    if (nlen == 0)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    *end = atoi(pNumber);
    p = pNumber + nlen;

    if (p == pEnd) // default step is set to 1
    {
        *step = 1;
        return 0;
    }

    if (!(*p == ' ' || *p == '\t'))
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }
	
    if (!(memcmp(p, _PREPROCESS_TAG_STR_FOR_STEP,
                    _PREPROCESS_TAG_LEN_FOR_STEP) == 0 &&
                (*(p+_PREPROCESS_TAG_LEN_FOR_STEP) == ' ' ||
                 *(p+_PREPROCESS_TAG_LEN_FOR_STEP) == '\t')))
    {
		logWarning("file: "__FILE__", line: %d, "
                "unkown for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    p += _PREPROCESS_TAG_LEN_FOR_STEP + 1;
    pNumber = iniGetInteger(p, pEnd, &nlen);
	
    if (nlen == 0)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }
	
    *step = atoi(pNumber);
    p = pNumber + nlen;
	
    while (p < pEnd && (*p == ' ' || *p == '\t'))
    {
        p++;
    }
	
    if (p != pEnd)
    {
		logWarning("file: "__FILE__", line: %d, "
                "invalid for range: %.*s", __LINE__,
                range_len, range);
        return EINVAL;
    }

    return 0;
}



// content is the conf file content which has been preProcessed
// return 0: success
static int iniDoLoadItemsFromBuffer(char *content, IniContext *pContext)
{
    AnnotationMap *pAnnoMap;
	IniSection *pSection;
	IniItem *pItem;
	char *pLine;
	char *pLastEnd;
	char *pEqualChar;
    char *pItemName;
    char *pAnnoItemLine;
	char *pIncludeFilename;
    char *pItemValues[100];
    char pFuncName[FAST_INI_ITEM_NAME_LEN + 1];
	char full_filename[MAX_PATH_SIZE];
    int i;
	int nLineLen;
	int nNameLen;
    int nItemCnt;
	int nValueLen;
	int result;
    int isAnnotation;

	result = 0;
    pAnnoItemLine = NULL;
    isAnnotation = 0;
    *pFuncName = '\0';
	pLastEnd = content - 1;
	pSection = pContext->current_section;
    pItem = pSection->items + pSection->count;

	while (pLastEnd != NULL)
	{
		pLine = pLastEnd + 1;  // pointer to the begin of a line
		pLastEnd = strchr(pLine, '\n'); // pointer to the end of a line
		
		if (pLastEnd != NULL)
		{
			*pLastEnd = '\0';
		}

        if (isAnnotation && pLine != pAnnoItemLine)
        {
            logWarning("file: "__FILE__", line: %d, " \
                "the @function annotation line " \
                "must follow by key=value line!", __LINE__);
            isAnnotation = 0;
        }

		if (*pLine == '#' && \
			strncasecmp(pLine+1, "include", 7) == 0 && \
			(*(pLine+8) == ' ' || *(pLine+8) == '\t'))
		{
			pIncludeFilename = strdup(pLine + 9);
			
			if (pIncludeFilename == NULL)
			{
				logError("file: "__FILE__", line: %d, " \
					"strdup %d bytes fail", __LINE__, \
					(int)strlen(pLine + 9) + 1);
				result = errno != 0 ? errno : ENOMEM;
				break;
			}

			trim(pIncludeFilename);
			
			if (strncasecmp(pIncludeFilename, "http://", 7) == 0)
			{
				snprintf(full_filename, sizeof(full_filename),\
					"%s", pIncludeFilename);
			}
			else
			{
				if (*pIncludeFilename == '/')
				{
				snprintf(full_filename, sizeof(full_filename), \
					"%s", pIncludeFilename);
				}
				else
				{
				snprintf(full_filename, sizeof(full_filename), \
					"%s/%s", pContext->config_path, \
					 pIncludeFilename);
				}

				if (!fileExists(full_filename))
				{
					logError("file: "__FILE__", line: %d, " \
						"include file \"%s\" not exists, " \
						"line: \"%s\"", __LINE__, \
						pIncludeFilename, pLine);
					free(pIncludeFilename);
					result = ENOENT;
					break;
				}

				
			}

            pContext->current_section = &pContext->global;
			result = iniDoLoadFromFile(full_filename, pContext); // call again
			
			if (result != 0)
			{
				free(pIncludeFilename);
				break;
			}

            pContext->current_section = &pContext->global; // 将当前处理的section指向pContext->global
			pSection = pContext->current_section;
            pItem = pSection->items + pSection->count;  //must re-asign

			free(pIncludeFilename);
			continue;
		}
        else if ((*pLine == '#' && \
            strncasecmp(pLine+1, "@function", 9) == 0 && \
            (*(pLine+10) == ' ' || *(pLine+10) == '\t')))  // 是否是@function annotation
        {
            if (!pContext->ignore_annotation) 
			{
                nNameLen = strlen(pLine + 11);
				
                if (nNameLen > FAST_INI_ITEM_NAME_LEN)
                {
                    nNameLen = FAST_INI_ITEM_NAME_LEN;
                }
				
                memcpy(pFuncName, pLine + 11, nNameLen);
                pFuncName[nNameLen] = '\0';
                trim(pFuncName);
				
                if ((int)strlen(pFuncName) > 0)
                {
                    isAnnotation = 1;
                    pAnnoItemLine = pLastEnd + 1;
                }
                else
                {
                    logWarning("file: "__FILE__", line: %d, " \
                            "the function name of annotation line is empty", \
                            __LINE__);
                }
				
            }
			
            continue;
			
        }

		trim(pLine);
		
		if (*pLine == '#' || *pLine == '\0') 
		{
			continue;
		}

		nLineLen = strlen(pLine);
		
		if (*pLine == '[' && *(pLine + (nLineLen - 1)) == ']') //section
		{
			char *section_name;
			int section_len;

			*(pLine + (nLineLen - 1)) = '\0';
			section_name = pLine + 1; //skip [

			trim(section_name);
			
			if (*section_name == '\0') //global section
			{
				pContext->current_section = &pContext->global;
				pSection = pContext->current_section;
                pItem = pSection->items + pSection->count;
				continue;
			}

			section_len = strlen(section_name);
			pSection = (IniSection *)hash_find(&pContext->sections,\
					section_name, section_len);
			
			if (pSection == NULL)  // not find
			{
				pSection = (IniSection *)malloc(sizeof(IniSection));
				
				if (pSection == NULL)
				{
					result = errno != 0 ? errno : ENOMEM;
					logError("file: "__FILE__", line: %d, "\
						"malloc %d bytes fail, " \
						"errno: %d, error info: %s", \
						__LINE__, \
						(int)sizeof(IniSection), \
						result, STRERROR(result));

					break;
				}

				memset(pSection, 0, sizeof(IniSection));
				result = hash_insert(&pContext->sections, \
					  section_name, section_len, pSection);
				
				if (result < 0)
				{
					result *= -1;
					logError("file: "__FILE__", line: %d, "\
						"insert into hash table fail, "\
						"errno: %d, error info: %s", \
						__LINE__, result, \
						STRERROR(result));
					break;
				}
				else
				{
					result = 0;
				}
			}

			pContext->current_section = pSection; 
                        pItem = pSection->items + pSection->count;
			continue;
		}

		pEqualChar = strchr(pLine, '=');
		
		if (pEqualChar == NULL)
		{
			continue;
		}

		nNameLen = pEqualChar - pLine;
		nValueLen = strlen(pLine) - (nNameLen + 1); // means we can not have ' ' or '\t' before '=' and after '='
		
		if (nNameLen > FAST_INI_ITEM_NAME_LEN)
		{
			nNameLen = FAST_INI_ITEM_NAME_LEN;
		}

		if (nValueLen > FAST_INI_ITEM_VALUE_LEN)
		{
			nValueLen = FAST_INI_ITEM_VALUE_LEN;
		}

		if (pSection->count >= pSection->alloc_count)
        {
            result = remallocSection(pSection, &pItem);
			
            if (result)
            {
                break;
            }
		}

		memcpy(pItem->name, pLine, nNameLen);
		memcpy(pItem->value, pEqualChar + 1, nValueLen); // means we can not have ' ' or '\t' before '=' and after '='

		trim(pItem->name);  // it allow has space before or after '='
		trim(pItem->value);

        if (isAnnotation) // access the global variable: g_annotataionMap and make some process
        {
            isAnnotation = 0;

            if (g_annotataionMap == NULL)
            {
                logWarning("file: "__FILE__", line: %d, " \
                    "not set annotataionMap and (%s) will use " \
                    "the item value (%s)", __LINE__, pItem->name,
                    pItem->value);
				
                pSection->count++;
                pItem++;
                continue;
            }

            nItemCnt = -1;
            pAnnoMap = g_annotataionMap;
			
            while (pAnnoMap->func_name)
            {
                if (strcmp(pFuncName, pAnnoMap->func_name) == 0)
                {
                    if (pAnnoMap->func_init)
                    {
                        pAnnoMap->func_init();
                    }

                    if (pAnnoMap->func_get)
                    {
                        nItemCnt = pAnnoMap->func_get(pItem->value, pItemValues, 100);
                    }
					
                    break;
                }
				
                pAnnoMap++;
            }

            if (nItemCnt == -1)
            {
                logWarning("file: "__FILE__", line: %d, " \
                    "not found corresponding annotation function: %s, " \
                    "\"%s\" will use the item value \"%s\"", __LINE__,
                    pFuncName, pItem->name, pItem->value);
                pSection->count++;
                pItem++;
                continue;
            }
            else if (nItemCnt == 0)
            {
                logWarning("file: "__FILE__", line: %d, " \
                    "annotation function %s execute fail, " \
                    "\"%s\" will use the item value \"%s\"", __LINE__,
                    pFuncName, pItem->name, pItem->value);
                pSection->count++;
                pItem++;
                continue;
            }

            pItemName = pItem->name;
            nNameLen = strlen(pItemName);
			
            for (i = 0; i < nItemCnt; i++)
            {
                nValueLen = strlen(pItemValues[i]);
				
                if (nValueLen > FAST_INI_ITEM_VALUE_LEN)
                {
                    nValueLen = FAST_INI_ITEM_VALUE_LEN;
                }
				
                memcpy(pItem->name, pItemName, nNameLen);
                memcpy(pItem->value, pItemValues[i], nValueLen);
                pItem->value[nValueLen] = '\0';
                pSection->count++;
                pItem++;
				
                if (pSection->count >= pSection->alloc_count)
                {
                    result = remallocSection(pSection, &pItem);
					
                    if (result)
                    {
                        break;
                    }
                }
				
            }
            continue;
        }

		pSection->count++;
		pItem++;
	}


    // result is 0 means has not error occured
    
    if (result == 0 && isAnnotation)
    {
        logWarning("file: "__FILE__", line: %d, " \
            "the @function annotation line " \
            "must follow by key=value line!", __LINE__);
    }

	return result; 
}

// get the rid of white space from the left and right of pStr
char *trim(char *pStr)
{
	trim_right(pStr);
	trim_left(pStr);
	return pStr;
}

//get rid of the white space from the left of pStr
char *trim_left(char *pStr)
{
	char *p;
	char *pEnd;
	int nDestLen;

	pEnd = pStr + strlen(pStr);
	
	for (p=pStr; p<pEnd; p++)
	{
		if (!(' ' == *p|| '\n' == *p || '\r' == *p || '\t' == *p))
		{
			break;
		}
	}
	
	if ( p == pStr)
	{
		return pStr;
	}
	
	nDestLen = (pEnd - p) + 1; //including \0
	memmove(pStr, p, nDestLen);

	return pStr;
}


// get rid of the white space from the right of pStr
char *trim_right(char *pStr)
{
	int len;
	char *p;
	char *pEnd;

	len = strlen(pStr);
	
	if (len == 0)
	{
		return pStr;
	}

	pEnd = pStr + len - 1;
	
	for (p = pEnd;  p>=pStr; p--)
	{
		if (!(' ' == *p || '\n' == *p || '\r' == *p || '\t' == *p))
		{
			break;
		}
	}

	if (p != pEnd)
	{
		*(p+1) = '\0';
	}

	return pStr;
}


bool fileExists(const char *filename)
{
	return access(filename, 0) == 0;
}

// allocate enough space for section
static int remallocSection(IniSection *pSection, IniItem **pItem)
{
    int bytes, result;
    IniItem *pNew;

    if (pSection->alloc_count == 0)
    {
        pSection->alloc_count = _INIT_ALLOC_ITEM_COUNT;
    }
    else
    {
        pSection->alloc_count *= 2;
    }
	
    bytes = sizeof(IniItem) * pSection->alloc_count;
    pNew = (IniItem *)malloc(bytes);
	
    if (pNew == NULL)
    {
        logError("file: "__FILE__", line: %d, " \
            "malloc %d bytes fail", __LINE__, bytes);
        result = errno != 0 ? errno : ENOMEM;
        return result;
    }

    if (pSection->count > 0)
    {
        memcpy(pNew, pSection->items,
                sizeof(IniItem) * pSection->count);
        free(pSection->items);
    }

    pSection->items = pNew;
    *pItem = pSection->items + pSection->count;
	
    memset(*pItem, 0, sizeof(IniItem) * \
        (pSection->alloc_count - pSection->count));

    return 0;
}

void *hash_find(HashArray *pHash, const void *key, const int key_len)
{
	unsigned int hash_code;
	HashData **ppBucket;
	HashData *hash_data;

	hash_code = pHash->hash_func(key, key_len);
	ppBucket = pHash->buckets + (hash_code % (*pHash->capacity));

	HASH_LOCK(pHash, ppBucket - pHash->buckets)
	hash_data = _chain_find_entry(ppBucket, key, key_len, hash_code);
	HASH_UNLOCK(pHash, ppBucket - pHash->buckets)

	if (hash_data != NULL)
	{
		return hash_data->value;
	}
	else
	{
		return NULL;
	}
}

#define HASH_LOCK(pHash, index) \
	if (pHash->lock_count > 0) \
	{ \
		pthread_mutex_lock(pHash->locks + (index) % pHash->lock_count); \
	}

#define HASH_UNLOCK(pHash, index) \
	if (pHash->lock_count > 0) \
	{ \
		pthread_mutex_unlock(pHash->locks + (index) % pHash->lock_count); \
	}

static HashData *_chain_find_entry(HashData **ppBucket, const void *key, \
		const int key_len, const unsigned int hash_code)
{
	HashData *hash_data;

	hash_data = *ppBucket;
	
	while (hash_data != NULL)
	{
		if (key_len == hash_data->key_len && \
			memcmp(key, hash_data->key, key_len) == 0)
		{
			return hash_data;
		}

		hash_data = hash_data->next;
	}

	return NULL;
}

#define hash_insert(pHash, key, key_len, value) \
	hash_insert_ex(pHash, key, key_len, value, 0, true)

int hash_insert_ex(HashArray *pHash, const void *key, const int key_len, \
		void *value, const int value_len, const bool needLock)
{
	unsigned int hash_code;
	HashData **ppBucket;
	HashData *hash_data;
	HashData *previous;
	char *pBuff;
	int bytes;
	int malloc_value_size;

	hash_code = pHash->hash_func(key, key_len);
	ppBucket = pHash->buckets + (hash_code % (*pHash->capacity));

	previous = NULL;

	if (needLock)
	{
		HASH_LOCK(pHash, ppBucket - pHash->buckets)
	}

	hash_data = *ppBucket;
	
	while (hash_data != NULL)
	{
		if (key_len == hash_data->key_len && \
			memcmp(key, hash_data->key, key_len) == 0)
		{
			break;
		}

		previous = hash_data;
		hash_data = hash_data->next;
		
	}

	if (hash_data != NULL) //exists
	{
		if (!pHash->is_malloc_value)
		{
			hash_data->value_len = value_len;
			hash_data->value = (char *)value;
			
			if (needLock)
			{
				HASH_UNLOCK(pHash, ppBucket - pHash->buckets)
			}
			
			return 0;
		}
		else
		{
			if (hash_data->malloc_value_size >= value_len && \
				(hash_data->malloc_value_size <= 128 ||
				 hash_data->malloc_value_size / 2 < value_len))
			{
				hash_data->value_len = value_len;
				memcpy(hash_data->value, value, value_len);
				
				if (needLock)
				{
					HASH_UNLOCK(pHash, ppBucket - pHash->buckets)
				}
				
				return 0;
			}

			DELETE_FROM_BUCKET(pHash, ppBucket, previous, hash_data)
		}
	}
	
	if (needLock)
	{
		HASH_UNLOCK(pHash, ppBucket - pHash->buckets)
	}

	if (!pHash->is_malloc_value)
	{
		malloc_value_size = 0;
	}
	else
	{
		malloc_value_size = MEM_ALIGN(value_len);
	}

	bytes = CALC_NODE_MALLOC_BYTES(key_len, malloc_value_size);
	
	if (pHash->max_bytes > 0 && pHash->bytes_used+bytes > pHash->max_bytes)
	{
		return -ENOSPC;
	}

	pBuff = (char *)malloc(bytes);
	
	if (pBuff == NULL)
	{
		return -ENOMEM;
	}

	pHash->bytes_used += bytes;

	hash_data = (HashData *)pBuff;
	hash_data->malloc_value_size = malloc_value_size;

	hash_data->key_len = key_len;
	memcpy(hash_data->key, key, key_len);
#ifdef HASH_STORE_HASH_CODE
	hash_data->hash_code = hash_code;  // save the hash code for next use, it need not calculate again when next use
#endif
	hash_data->value_len = value_len;

	if (!pHash->is_malloc_value) // means the buffer of value is allocated in outer space, rather than beening allocated in the end of the key buffer
	{
		hash_data->value = (char *)value;
	}
	else
	{
		hash_data->value = hash_data->key + hash_data->key_len;
		memcpy(hash_data->value, value, value_len);
	}

	if (needLock)
	{
		HASH_LOCK(pHash, ppBucket - pHash->buckets)
		ADD_TO_BUCKET(pHash, ppBucket, hash_data)
		HASH_UNLOCK(pHash, ppBucket - pHash->buckets)
	}
	else
	{
		ADD_TO_BUCKET(pHash, ppBucket, hash_data)
	}

	if (pHash->load_factor >= 0.10 && (double)pHash->item_count /
		(double)*pHash->capacity >= pHash->load_factor)
	{
		_rehash(pHash);  // rehash 
	}

	return 1;
}

// delete a hash_data from pHash
#define DELETE_FROM_BUCKET(pHash, ppBucket, previous, hash_data) \
	if (previous == NULL) \
	{ \
		*ppBucket = hash_data->next; \
	} \
	else \
	{ \
		previous->next = hash_data->next; \
	} \
	pHash->item_count--; \
	pHash->bytes_used -= CALC_NODE_MALLOC_BYTES(hash_data->key_len, \
				hash_data->malloc_value_size); \
	free(hash_data);


#define MEM_ALIGN(x)  (((x) + 7) & (~7))

#define CALC_NODE_MALLOC_BYTES(key_len, value_size) \
		sizeof(HashData) + key_len + value_size


#define ADD_TO_BUCKET(pHash, ppBucket, hash_data) \
	hash_data->next = *ppBucket; \
	*ppBucket = hash_data; \
	pHash->item_count++;

static int _rehash(HashArray *pHash)
{
	int result;
	unsigned int *pOldCapacity;

	pOldCapacity = pHash->capacity;
	
	if (pHash->is_malloc_capacity)
	{
		unsigned int *pprime;
		unsigned int *prime_end;

		pHash->capacity = NULL;

		prime_end = prime_array + PRIME_ARRAY_SIZE;
		
		for (pprime = prime_array; pprime!=prime_end; pprime++)
		{
			if (*pprime > *pOldCapacity)
			{
				pHash->capacity = pprime;
				break;
			}
		}
	}
	else
	{
		pHash->capacity++;
	}

	if ((result=_rehash1(pHash, *pOldCapacity, pHash->capacity)) != 0)
	{
		pHash->capacity = pOldCapacity;  //rollback
	}
	else
	{
		if (pHash->is_malloc_capacity)
		{
			free(pOldCapacity);
			pHash->is_malloc_capacity = false;
		}
	}

	/*printf("rehash, old_capacity=%d, new_capacity=%d\n", \
		old_capacity, *pHash->capacity);
	*/
	return result;
}


// rehash by new size: *new_capacity
static int _rehash1(HashArray *pHash, const int old_capacity, \
		unsigned int *new_capacity)
{
	HashData **old_buckets;
	HashData **ppBucket;
	HashData **bucket_end;
	HashData *hash_data;
	HashData *pNext;
	int result;

	old_buckets = pHash->buckets;
	pHash->capacity = new_capacity;
	
	if ((result=_hash_alloc_buckets(pHash, old_capacity)) != 0)
	{
		pHash->buckets = old_buckets;
		return result;
	}

	//printf("old: %d, new: %d\n", old_capacity, *pHash->capacity);

	pHash->item_count = 0;
	bucket_end = old_buckets + old_capacity;
	
	for (ppBucket=old_buckets; ppBucket<bucket_end; ppBucket++)
	{
		if (*ppBucket == NULL)
		{
			continue;
		}

		hash_data = *ppBucket;
		
		while (hash_data != NULL)
		{
			pNext = hash_data->next;

			ADD_TO_BUCKET(pHash, (pHash->buckets + \
				(HASH_CODE(pHash, hash_data) % \
				(*pHash->capacity))), hash_data)

			hash_data = pNext;
		}
	}

	free(old_buckets);
	return 0;
}


char *iniGetStrValue(const char *szSectionName, const char *szItemName, \
		IniContext *pContext)
{
	IniItem targetItem;
	IniSection *pSection;
	IniItem *pItem;

	INI_FIND_ITEM(szSectionName, szItemName, pContext, pSection, \
			targetItem, pItem, NULL)

	if (pItem == NULL)
	{
		return NULL;
	}
	else
	{
		return pItem->value;
	}
}

#define INI_FIND_ITEM(szSectionName, szItemName, pContext, pSection, \
			targetItem, pItem, return_val) \
	if (szSectionName == NULL || *szSectionName == '\0') \
	{ \
		pSection = &pContext->global; \
	} \
	else \
	{ \
		pSection = (IniSection *)hash_find(&pContext->sections, \
				szSectionName, strlen(szSectionName)); \
		if (pSection == NULL) \
		{ \
			return return_val; \
		} \
	} \
	\
	if (pSection->count <= 0) \
	{ \
		return return_val; \
	} \
	\
	snprintf(targetItem.name, sizeof(targetItem.name), "%s", szItemName); \
	pItem = (IniItem *)bsearch(&targetItem, pSection->items, \
			pSection->count, sizeof(IniItem), iniCompareByItemName); // 二分查找



static void iniSortItems(IniContext *pContext)
{
	if (pContext->global.count > 1)
	{
		qsort(pContext->global.items, pContext->global.count, \
			sizeof(IniItem), iniCompareByItemName);
	}

	hash_walk(&pContext->sections, iniSortHashData, NULL);
}


static int iniCompareByItemName(const void *p1, const void *p2)
{
	return strcmp(((IniItem *)p1)->name, ((IniItem *)p2)->name);
}

int hash_walk(HashArray *pHash, HashWalkFunc walkFunc, void *args)
{
	HashData **ppBucket;
	HashData **bucket_end;
	HashData *hash_data;
	int index;
	int result;

	index = 0;
	bucket_end = pHash->buckets + (*pHash->capacity);
	
	for (ppBucket=pHash->buckets; ppBucket<bucket_end; ppBucket++)
	{
		hash_data = *ppBucket;
		
		while (hash_data != NULL)
		{
			result = walkFunc(index, hash_data, args);
			
			if (result != 0)
			{
				return result;
			}

			index++;
			hash_data = hash_data->next;
		}
	}

	return 0;
}

void iniFreeContext(IniContext *pContext)
{
	if (pContext == NULL)
	{
		return;
	}

	if (pContext->global.items != NULL)
	{
		free(pContext->global.items);
		memset(&pContext->global, 0, sizeof(IniSection));
	}

	hash_walk(&pContext->sections, iniFreeHashData, NULL);
	hash_destroy(&pContext->sections);

    iniFreeDynamicContent(pContext);
}

void hash_destroy(HashArray *pHash)
{
	HashData **ppBucket;
	HashData **bucket_end;
	HashData *pNode;
	HashData *pDelete;

	if (pHash == NULL || pHash->buckets == NULL)
	{
		return;
	}

	bucket_end = pHash->buckets + (*pHash->capacity);
	
	for (ppBucket=pHash->buckets; ppBucket<bucket_end; ppBucket++)
	{
		pNode = *ppBucket;
		
		while (pNode != NULL)
		{
			pDelete = pNode;
			pNode = pNode->next;
			free(pDelete);
		}
	}

	free(pHash->buckets);
	pHash->buckets = NULL;
	
	if (pHash->is_malloc_capacity)
	{
		free(pHash->capacity);
		pHash->capacity = NULL;
		pHash->is_malloc_capacity = false;
	}

	pHash->item_count = 0;
	pHash->bytes_used = 0;
}

static void iniFreeDynamicContent(IniContext *pContext)
{
    CDCPair *pCDCPair;
    DynamicContents *pDynamicContents;
    int i;

    if (g_dynamic_content_count == 0)
    {
        return;
    }

    if (g_dynamic_contents[g_dynamic_content_index].context == pContext)
    {
        pCDCPair = g_dynamic_contents + g_dynamic_content_index;
    }
    else
    {
        pCDCPair = NULL;
		
        for (i=0; i<_MAX_DYNAMIC_CONTENTS; i++)
        {
            if (g_dynamic_contents[i].context == pContext)
            {
                pCDCPair = g_dynamic_contents + i;
                break;
            }
        }
		
        if (pCDCPair == NULL)
        {
            return;
        }
    }

    pCDCPair->used = false;
    pCDCPair->context = NULL;
    pDynamicContents = &pCDCPair->dynamicContents;
	
    if (pDynamicContents->contents != NULL)
    {
        for (i=0; i<pDynamicContents->count; i++)
        {
            if (pDynamicContents->contents[i] != NULL)
            {
                free(pDynamicContents->contents[i]);
            }
        }
		
        free(pDynamicContents->contents);
        pDynamicContents->contents = NULL;
    }
	
    pDynamicContents->alloc_count = 0;
    pDynamicContents->count = 0;
    g_dynamic_content_count--;
}
```















