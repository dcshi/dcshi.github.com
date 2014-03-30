---
layout: post
status: publish
published: true
title: redis学习笔记之initServer
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 43
wordpress_url: http://www.dcshi.com/?p=43
date: '2011-10-15 10:47:56 +0000'
date_gmt: '2011-10-15 10:47:56 +0000'
categories:
- redis
tags:
- redis
- "笔记"
---
<p align="left">初始化 list列表</p>
<p align="left">server.clients = listCreate();<br />
server.slaves = listCreate();<br />
server.monitors = listCreate();</p>
<p align="left">server.unblocked_clients = listCreate();</p>
<p align="left">创建shared对象，create a number of shared objects, which are accessible via the global shared struct.<br />
<strong>createSharedObjects</strong>();</p>
<p align="left">struct sharedObjectsStruct {<br />
robj *crlf, *ok, *err, *emptybulk, *czero, *cone, *cnegone, *pong, *space,<br />
*colon, *nullbulk, *nullmultibulk, *queued,<br />
*emptymultibulk, *wrongtypeerr, *nokeyerr, *syntaxerr, *sameobjecterr,<br />
*outofrangeerr, *loadingerr, *plus,<br />
*select0, *select1, *select2, *select3, *select4,<br />
*select5, *select6, *select7, *select8, *select9,<br />
*messagebulk, *pmessagebulk, *subscribebulk, *unsubscribebulk, *mbulk3,<br />
*mbulk4, *psubscribebulk, *punsubscribebulk,<br />
*integers[REDIS_SHARED_INTEGERS];<br />
};</p>
<p align="left">typedef struct redisObject {<br />
unsigned type:4;<br />
unsigned storage:2;     /* REDIS_VM_MEMORY or REDIS_VM_SWAPPING */<br />
unsigned encoding:4;<br />
unsigned lru:22;        /* lru time (relative to server.lruclock) */<br />
int refcount;<br />
void *ptr;<br />
/* VM fields are only allocated if VM is active, otherwise the<br />
* object allocation function will just allocate<br />
* sizeof(redisObjct) minus sizeof(redisObjectVM), so using<br />
* Redis without VM active will not have any overhead. */<br />
} robj;</p>
<p align="left">创建epoll实例，为aeEventLoop结构体分配空间，对它的一些fields做一个简单的初始化<br />
server.el = <strong>aeCreateEventLoop</strong>();</p>
<p align="left">/* State of an event based program */<br />
typedef struct <strong>aeEventLoop</strong> {<br />
int maxfd;<br />
long long timeEventNextId;<br />
aeFileEvent events[AE_SETSIZE];</p>
<p align="left">/*</p>
<p align="left">在epoll中注册了的事件也需要在此初始化一个记录，其实就对应一个句柄，</p>
<p align="left">标注这个句柄状态（读写）发生变化的时候的处理函数（writeProc/readProc）,</p>
<p align="left">mask表示这个句柄等待发生的事件（读/写）,</p>
<p align="left">clientData是这个句柄对应的要接收的/发送的数据，其实就是一个数据缓冲区</p>
<p align="left">typedef struct aeFileEvent {<br />
int mask; /* one of AE_(READABLE|WRITABLE) */<br />
aeFileProc *rfileProc;<br />
aeFileProc *wfileProc;<br />
void *clientData;<br />
} aeFileEvent;</p>
<p align="left">*/<br />
aeFiredEvent fired[AE_SETSIZE];</p>
<p align="left">/*</p>
<p align="left">状态发生了的事件都在这里注册一份，</p>
<p align="left">这样是为了serverCron（一个timeEevnt类型的计划任务）迅速找到这些待处理的事件</p>
<p align="left">然后根据fd结合上面提到的aeFileEvnt中找到相应的处理函数与数据</p>
<p align="left">ServerCron其实就是一个异步处理的思想</p>
<p align="left">typedef struct aeFiredEvent {<br />
int fd;<br />
int mask;<br />
} aeFiredEvent;</p>
<p align="left">*/<br />
aeTimeEvent *timeEventHead;<br />
int stop;<br />
void *apidata;</p>
<p align="left">/*这个是给epoll用的，数据结构如下：</p>
<p align="left">typedef struct aeApiState {</p>
<p align="left">int epfd;<br />
struct epoll_event events[AE_SETSIZE];</p>
<p align="left">} aeApiState;</p>

<p align="left">events是当前状态发生改变的事件集合</p>
<p align="left">*/</p>
<p align="left">aeBeforeSleepProc *beforesleep;<br />
} aeEventLoop;</p>


<p align="left">//linux系统对FD的处理：</p>
<p align="left">1.首先fd是进程相关的，fd只是一个数字，每个进程都可以有重复的这些数字</p>
<p align="left">2.假设当前server只能打开的files数目是1024，那fd的分布只能是1-1024.</p>
<p align="left">3.fd的分配是从当前可用的最小的fd开始分配的，假设 打开的fd为 6 7 8，如果close(7),下次分配的fd应该是7而不是9</p>


<p align="left"><strong>anetTcpServer</strong></p>
<p align="left">//就是简单的TCPserver初始化，你能想到的socket初始化，setscoketopt，bind，listen操作都在里面</p>


<p align="left"><strong>server DB</strong><strong>初始化</strong></p>
<p align="left">for (j = 0; j &lt; server.dbnum; j++) {<br />
server.db[j].dict = dictCreate(&amp;dbDictType,NULL);<br />
server.db[j].expires = dictCreate(&amp;keyptrDictType,NULL);<br />
server.db[j].blocking_keys = dictCreate(&amp;keylistDictType,NULL);<br />
server.db[j].watched_keys = dictCreate(&amp;keylistDictType,NULL);<br />
if (server.vm_enabled)<br />
server.db[j].io_keys = dictCreate(&amp;keylistDictType,NULL);<br />
server.db[j].id = j;<br />
}</p>
<p align="left"><strong>redisDb</strong><strong>介绍：</strong></p>
<p align="left">typedef struct redisDb {</p>
<p align="left">dict *dict;                 /* The keyspace for this DB */<br />
dict *expires;              /* Timeout of keys with a timeout set */<br />
dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */<br />
dict *io_keys;              /* Keys with clients waiting for VM I/O */<br />
dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */<br />
int id;<br />
} redisDb;</p>


<p align="left"><strong>List</strong><strong>数据类型：</strong></p>
<p align="left">server.clients = listCreate();<br />
server.slaves = listCreate();<br />
server.monitors = listCreate();</p>
<p align="left">server.unblocked_clients = listCreate();</p>
<p align="left">server.pubsub_patterns = listCreate(); /* A list of pubsub_patterns */<br />
listSetFreeMethod(server.pubsub_patterns,freePubsubPattern);<br />
listSetMatchMethod(server.pubsub_patterns,listMatchPubsubPattern);</p>
<p align="left">typedef struct list {</p>
<p align="left">listNode *head;<br />
listNode *tail;<br />
void *(*dup)(void *ptr);<br />
void (*free)(void *ptr);<br />
int (*match)(void *ptr, void *key);<br />
unsigned int len;</p>
<p align="left">} list;</p>

<p align="left">typedef struct listNode {</p>
<p align="left">struct listNode *prev;<br />
struct listNode *next;<br />
void *value;</p>
<p align="left">} listNode;</p>

<p align="left"><strong>aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);</strong></p>
<p align="left">1)通过初始化一个</p>
<p align="left"><strong>aeTimeEvent</strong><strong>结构，</strong></p>
<p align="left">2)并操作eventLoop-&gt;timeEventHead把新来的timeEvent事件插到对头</p>
<p align="left">serverCron是一个计划任务，也是Redis最重要的一个timeEvent</p>
<p align="left">serverCron处理的任务包括：</p>
<p align="left">a)updateLRUClock calculating LRU information</p>
<p align="left">b)处理shutdown事件</p>
<p align="left">c)记录当前info到log</p>
<p align="left">d)tryResizeHashTables 当有子进程在执行数据保存操作的时候，hashtable不会进行resize操作</p>
<p align="left">e)Show information about connected clients</p>
<p align="left">f)Close connections of timedout clients</p>
<p align="left">g)保存数据：前台执行bgsave命令或者后台判断是否已经达到自动保存条件</p>
<p align="left">h)activeExpireCycle 清零超时key，这只会在master上执行，slave上不会执行</p>
<p align="left">i)如果内存达到上限并且有启用虚拟内存的功能，把内存数据warp到虚拟内存</p>
<p align="left">j)replicationCron  主从同步</p>


<p align="left"><strong>aeCreateFileEvent(server.el,server.ipfd,AE_READABLE, acceptTcpHandler,NULL)</strong></p>
<p align="left">把anetTcpServer返回的fd，注册到epoll的事件中，设置只读属性，一旦发现fd可读，调用acceptTcpHandler函数，其实就是一个接收请求的功能</p>

<p align="left"><strong>acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask)</strong></p>
<p align="left">a)接收请求，并把accept后的fd注册到epoll事件</p>
<p align="left">b)创建一个redisClient客户端</p>
<p align="left">c)插入到server.clients list中</p>

<p align="left"><strong>vmInit()</strong></p>
<p align="left">如果设置了启动虚拟内存，这里会做一个初始化操作。</p>



<p>&nbsp;</p>
