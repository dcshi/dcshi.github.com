---
layout: post
status: publish
published: true
title: redis事件库学习笔记
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 41
wordpress_url: http://www.dcshi.com/?p=41
date: '2011-10-15 10:47:24 +0000'
date_gmt: '2011-10-15 10:47:24 +0000'
categories:
- redis
tags:
- redis
- "笔记"
---
<p>&nbsp;</p>
<p>学习过程中做的笔记，比较乱，有兴趣的朋友欢迎交流<br />
sina微博：@dcshi twitter：@dcshi<br />
aeCreateEventLoop<br />
初始化eventLoop结构并调用aeApiCreate创建一个epoll句柄</p>
<p>createClient<br />
通过此函数可用初始化一个连接，<br />
并调用 aeCreateFileEvent创建一个FileEvent事件<br />
listAddNodeTail在client链表增加一个节点</p>
<p>aeCreateTimeEvent<br />
a)redisSetReplyObjectFunctions……<br />
b)通过初始化一个 aeTimeEvent结构，<br />
c)并操作eventLoop-&gt;timeEventHead把新来的timeEvent事件插到对头</p>
<p>epoll_wait 的timeout指的是最长等待时间，单位是毫秒</p>
<p>aeProcessEvents<br />
用来处理fileEvent和timeEvent<br />
假设在执行epoll_wait时，一直等不到文件事件触发，那岂不是程序就一直这样阻塞着，连后面的TimeEvent相关的代码都没机会运行了？<br />
解决这个问题有赖于epoll_wait函数可以接受一个参数，用来确定最长等待时间，如果在这段时间一直没有文件事件触发，epoll_wait不会傻傻等待，而会返回到用户空间。问题关键是如何确定这个最长等待时间：<br />
一轮循环内，在处理FileEvent之前，会先查找最近的TimeEvent（调用aeSearchNearestTimer</p>
<p>aeSearchNearestTimer<br />
通过便利比较<br />
typedef struct aeTimeEvent<br />
{<br />
long long id; /* time event identifier. */<br />
long when_sec; /* seconds */<br />
long when_ms; /* milliseconds */<br />
aeTimeProc *timeProc;<br />
aeEventFinalizerProc *finalizerProc;<br />
void *clientData;<br />
struct aeTimeEvent *next;<br />
} aeTimeEvent;<br />
中的when_sec和when_ms变量来找出最老的一个aeTimeEvent，并返回</p>
<p>aeApiPoll（处理fileEvent）<br />
返回状态改变了的句柄数<br />
通过epoll_wait,返回可操作句柄数，其中epoll_wait的第二个参数是一个结果存储参数，保存可用的epoll_event,他是一个数组</p>
<p>processTimeEvents<br />
首先看下aeTimeEvent的数据结构<br />
/* Time event structure */<br />
typedef struct aeTimeEvent {<br />
long long id; /* time event identifier. */<br />
long when_sec; /* seconds */<br />
long when_ms; /* milliseconds */<br />
aeTimeProc *timeProc;<br />
aeEventFinalizerProc *finalizerProc;<br />
void *clientData;<br />
struct aeTimeEvent *next;<br />
} aeTimeEvent;</p>
<p>timeProc是超时处理函数，</p>
<p>如果timeProc返回-1，即AE_NOMORE（-1），<br />
就删除timeEvent（调用aeDeleteTimeEvent）<br />
否则调用aeAddMillisecondsToNow更新时间</p>
<p>Q:如果老的timeEvents注册新的timeEvent，会造成死循环吗？<br />
A：不会的，新注册的timeEvent不会再当前的循环中立即执行，从代码可以看出：<br />
if (te-&gt;id &gt; maxId) {<br />
te = te-&gt;next;<br />
continue;<br />
}<br />
maxId是本次processTimeEvents处理前event的最大 ID</p>
<p>&nbsp;</p>
