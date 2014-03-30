---
layout: post
status: publish
published: true
title: redis 内存管理篇
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 15
wordpress_url: http://www.dcshi.com/?p=15
date: '2011-10-15 10:26:44 +0000'
date_gmt: '2011-10-15 10:26:44 +0000'
categories:
- server
- redis
tags:
- redis
- "内存"
---
<p align="left">转 <a href="http://www.cnblogs.com/kernel_hcy/archive/2011/05/15/2046963.html" target="_blank">http://www.cnblogs.com/kernel_hcy/archive/2011/05/15/2046963.html</a></p>
<p align="left">redis的内存管理只有两个文件zmalloc.c和zmalloc.h。<br />
zmalloc.h的内容如下：</p>
<p align="left">1 void *zmalloc(size_t size);</p>
<p align="left">2 void *zcalloc(size_t size);</p>
<p align="left">3 void *zrealloc(void *ptr, size_t size);</p>
<p align="left">4 void zfree(void *ptr);</p>
<p align="left">5 char *zstrdup(const char *s);</p>
<p align="left">6 size_t zmalloc_used_memory(void);</p>
<p align="left">7 void zmalloc_enable_thread_safeness(void);</p>
<p align="left">8 float zmalloc_get_fragmentation_ratio(void);</p>
<p align="left">9 size_t zmalloc_get_rss(void);</p>
<p align="left">10 size_t zmalloc_allocations_for_size(size_t size);</p>
<p align="left">11</p>
<p align="left">12 #define ZMALLOC_MAX_ALLOC_STAT 256</p>
<p align="left">就这么几行。这是redis的内存管理接口。zmalloc,zcalloc,zrealloc和zfree分别对应c库中的malloc,calloc,realloc和free。zstrdup用于生成一个字符串的拷贝。后面的几个函数用于获取内存使用信息，后面会详细介绍。<br />
在看zmalloc.c的源码之前，先看看redis是怎样管理内存的。redis为了方便内存的管理，在分配一块内存之后，会将这块内存的大小存入内存块的头部。如下图。<br />
real_ptr ret_ptr<br />
| |<br />
V V<br />
—————————————————————–<br />
| size | memory block |<br />
—————————————————————-<br />
real_ptr是redis调用malloc后返回的指针。redis将内存块的大小size存入头部，size所占据的内存大小是已知的，为size_t类型的长度，然后返回ret_ptr。当需要释放内存的时候，ret_ptr被传给内存管理程序。通过ret_ptr，程序可以很容易的算出real_ptr的值，然后将real_ptr传给free释放内存。<br />
redis会记录所有的内存分配情况。redis定义一个数组，这个数组的长度为ZMALLOC_MAX_ALLOC_STAT。数组的每一个元素代表当前程序所分配的内存块的个数，且内存块的大小为该元素的下标。在程序中，这个数组为zmalloc_allocations。zmalloc_allocations[16]代表已经分配的长度为16bytes的内存块的个数。zmalloc.c中有一个静态变量used_memory用来记录当前分配的内存总大小。<br />
下面开始分析代码。<br />
首先是定义宏PREFIX_SIZE：</p>
<p align="left">1 #ifdef HAVE_MALLOC_SIZE</p>
<p align="left">2  #define PREFIX_SIZE (0)</p>
<p align="left">3  #else</p>
<p align="left">4  #if defined(__sun)</p>
<p align="left">5  #define PREFIX_SIZE (sizeof(long long))</p>
<p align="left">6  #else</p>
<p align="left">7  #define PREFIX_SIZE (sizeof(size_t))</p>
<p align="left">8  #endif</p>
<p align="left">9  #endif</p>
<p align="left">如果定义了HAVE_MALLOC_SIZE，那么PREFIX_SIZE就为0。这里，HAVE_MALLOC_SIZE用来确定系统是否有函数malloc_size。这个HAVE_MALLOC_SIZE宏在config.h中定义，如下：</p>
<p align="left"> 1 /* Use tcmalloc's malloc_size() when available.</p>
<p align="left"> 2 * When tcmalloc is used, native OSX malloc_size() may never be used because</p>
<p align="left"> 3 * this expects a different allocation scheme. Therefore, *exclusively* use</p>
<p align="left"> 4 * either tcmalloc or OSX's malloc_size()! */</p>
<p align="left"> 5  #if defined(USE_TCMALLOC)</p>
<p align="left"> 6 #include</p>
<p align="left"> 7 #if TC_VERSION_MAJOR &gt;= 1 &amp;&amp; TC_VERSION_MINOR &gt;= 6</p>
<p align="left"> 8 #define HAVE_MALLOC_SIZE 1</p>
<p align="left"> 9 #define redis_malloc_size(p) tc_malloc_size(p)</p>
<p align="left">10 #endif</p>
<p align="left">11 #elif defined(__APPLE__)</p>
<p align="left">12 #include</p>
<p align="left">13 #define HAVE_MALLOC_SIZE 1</p>
<p align="left">14 #define redis_malloc_size(p) malloc_size(p)</p>
<p align="left">15 #endif</p>
<p align="left">如果使用google的tcmalloc库，那么，redis_malloc_size就对应与tcmalloc库的tc_malloc_size函数。如果是在apple的mac上编译，那么redis_malloc_size就对应与malloc_size。redis_malloc_size的功能是获得参数p所指向的内存块的大小。tcmalloc库在google的google-perftools库中，据说这个库在内存管理的效率上很惊艳。不过这个库是c++写的，而redis是c写的，两者揉一起还是有点不给力阿。。。<br />
如果没有malloc_size函数，那么在Solaris系统上，用long long类型的长度来定义PREFIX_SIZE，其他系统为size_t的长度。<br />
接着，定义下面这些宏。这些宏的作用是如果使用tcmalloc库，那么将库中的分配函数对应到标准库上。后面的函数可直接使用标准库函数的名称。在更换库的时候不需要更改。</p>
<p align="left">1 /* Explicitly override malloc/free etc when using tcmalloc. */<br />
2 #if defined(USE_TCMALLOC)<br />
3 #define malloc(size) tc_malloc(size)<br />
4 #define calloc(count,size) tc_calloc(count,size)<br />
5 #define realloc(ptr,size) tc_realloc(ptr,size)<br />
6 #define free(ptr) tc_free(ptr)<br />
7 #endif</p>
<p align="left">下面的两个宏用于更新zmalloc_allocations数组。update_zmalloc_stat_alloc用于在分配内存的时候更新已分配大小，update_zmalloc_stat_free用于在释放内存的时候删除对应的记录。</p>
<p align="left">1 #define update_zmalloc_stat_alloc(__n,__size) do { \<br />
2 size_t _n = (__n); \<br />
3 size_t _stat_slot = (__size &lt; ZMALLOC_MAX_ALLOC_STAT) ? __size : ZMALLOC_MAX_ALLOC_STAT; \<br />
4 if (_n&amp;(sizeof(long)-1)) _n += sizeof(long)-(_n&amp;(sizeof(long)-1)); \<br />
5 if (zmalloc_thread_safe) { \<br />
6 pthread_mutex_lock(&amp;used_memory_mutex); \<br />
7 used_memory += _n; \<br />
8 zmalloc_allocations[_stat_slot]++; \<br />
9 pthread_mutex_unlock(&amp;used_memory_mutex); \<br />
10 } else { \<br />
11 used_memory += _n; \<br />
12 zmalloc_allocations[_stat_slot]++; \<br />
13 } \<br />
14 } while(0)</p>
<p align="left">update_zmalloc_stat_alloc的第一个参数__n是从系统那实际获得的内存大小，第二个参数是程序请求的内存大小。update_zmalloc_stat_alloc首先判断程序请求的内存大小在zmalloc_allocations数组中对应的下标。如果内存大小大于zmalloc_allocations数组的长度-1，那么其对应的下标是最后一个。然后，将实际分配的内存大小对齐为long类型长度的整数倍（malloc通常会考虑对齐问题，实际分配的内存大小也会因对齐而有所出入，后文会介绍）。最后，在used_memory记录实际分配的大小，在zmalloc_allocations对应位置加一。这里如果设置为线程安全，那么在记录之前要对两个静态变量加锁。</p>
<p align="left">1 #define update_zmalloc_stat_free(__n) do { \<br />
2 size_t _n = (__n); \<br />
3 if (_n&amp;(sizeof(long)-1)) _n += sizeof(long)-(_n&amp;(sizeof(long)-1)); \<br />
4 if (zmalloc_thread_safe) { \<br />
5 pthread_mutex_lock(&amp;used_memory_mutex); \<br />
6 used_memory -= _n; \<br />
7 pthread_mutex_unlock(&amp;used_memory_mutex); \<br />
8 } else { \<br />
9 used_memory -= _n; \<br />
10 } \<br />
11 } while(0)</p>
<p align="left">update_zmalloc_stat_free和update_zmalloc_stat_alloc差不多，但仅仅减少了used_memory的值。<br />
对于zmalloc,zalloc,zrealloc和zfree这几个函数，仅仅是对标准库的函数的简单的封装。所做的工作除了调用标准库（也可能是tcmalloc库）的函数分配内存外，就是对每次分配和释放内存做合适的记录。如果系统中有malloc_size函数，那么直接调用前面的那两个宏，没什么可讲的。如果没有malloc_size函数，那么需要在所分配的内存头部的PREFIX_SIZE大小的区域内，记录内存块的大小。代码很简单，就一句：<br />
1 *((size_t*)ptr) = size;<br />
现将ptr转换成size_t类型的指针，然后将size的值赋给其指向的内存。笔者感觉没有必要在前面定义PREFIX_SIZE的时候区分系统，因为这里直接硬编码了内存大小的类型为size_t。前面的宏判断有点多此一举了。这里的size是返回的内存区域的大小，不包括保存大小的头部。<br />
读取内存块大小需要两步：<br />
1 realptr = (char*)ptr-PREFIX_SIZE;<br />
2 oldsize = *((size_t*)realptr);</p>
<p align="left">程序传进来的是ret_ptr，通过减去PREFIX_SIZE得到real_ptr。从real_ptr所指向的内存中读取大小即可。<br />
其余的几个函数也很直接，没什么好说的。最后讲一讲zmalloc_get_rss()函数。这个函数用来获取进程的RSS。神马是RSS？google reader那个？显然不是。。。全称为Resident Set Size，指实际使用物理内存（包含共享库占用的内存）。在linux系统中，可以通过读取/proc/pid/stat文件获取，pid为当前进程的进程号。读取到的不是byte数，而是内存页数。通过系统调用sysconf(_SC_PAGESIZE)可以获得当前系统的内存页大小。Unix系统貌似可以直接通过task_info直接获取，比linux系统简单的多。<br />
获得进程的RSS后，可以计算目前数据的内存碎片大小，直接用rss除以used_memory。rss包含进程的所有内存使用，包括代码，共享库，堆栈等。但是由于通常情况下redis在内存中数据的量要远远大于这些数据所占用的内存，因此这个简单的计算还是比较准确的。<br />
这里有一个问题，程序都是用多少内存就分配多少内存，哪来的内存碎片？其实，当调用malloc的时候，malloc并不是严格按照参数的值来分配内存。比如，程序只请求一个byte的内存，malloc不会就只分配一个byte，通常，基于内存对齐等方面的考虑，malloc会分配4个byte。这样，如果程序中大量请求1byte内存，那么实际使用的是所请求的4倍。malloc进行小内存分配是很浪费的。所以，碎片就在这里产生了。<br />
总的来说，redis的内存管理简单粗暴，没有神马复杂的引用计数等技术。但是，很多时候，简单的往往是高效且合理的。redis内存的中数据通常会是几个G，这个方法快速，统计结果也很精确。</p>

