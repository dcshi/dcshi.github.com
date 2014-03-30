---
layout: post
status: publish
published: true
title: "基于epoll的压力测试脚本"
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 5
wordpress_url: http://www.dcshi.com/?p=5
date: '2011-10-15 10:21:19 +0000'
date_gmt: '2011-10-15 10:21:19 +0000'
categories:
- "压力测试"
tags:
- "性能"
- "压力测试"
---
<p align="left">好吧，我们开始之前，如果我们要用多进程来做，会怎么来设计？<br />
1）初始化需要完成的预定请求数n（n是用户输入的参数）requests<br />
2）创建c（c是用户输入的参数）个worker进程（这里worker进程与主进程是父子关系，这样worker进程可以共享第一个步骤创建的requests变量）<br />
3）当一个worker进程完成一个请求以后，requests需要减一，这里涉及到变量安全，我们一般加一个锁，常用信号量(Semaphore)<br />
3.1)可能有同学对上面有不同想法，每个worerk进程完成一个请求要用用一次Semaphore效率很低，那样换个思路，每个worker进程都要一定的任务（例如，一共要完成10w个请求，10个worker，每个worker完成1w个请求后退出），由主进程来监控剩余的worker进程数目，如果为0，认为是所有请求完成。3和3.1的方案各有优缺，这里不作点评。</p>
<p align="left">今天主要介绍的是用epoll来完成压力测试工作。<br />
基于epoll而非多线程多进程，对于压力测试，除网络IO以外没有任何的磁盘IO等阻塞调用，个人认为用epoll完全可以模拟多进程的并发行为，而且实现更简单.但也有潜在问题，由于脚本是单进程模型，所以有可能测试脚本比测试对象先达到测试瓶颈（一般是CPU）</p>
<p align="left"><strong>代码</strong>：<a href="https://github.com/dcshi/benchmark" target="_blank">https://github.com/dcshi/benchmark/</a></p>
<p align="left"><strong>流程图：</strong></p>
<p align="left"><img class="alignnone" title="流程图" src="http://www.dcshi.com/wp-content/uploads/2011/10/benchmark.jpg" alt="" width="747" height="985" /></p>
<p align="left"><strong>脚本执行选项</strong><br />
Usage: benchmark [-H ] [-p ] [-c ] [-n ] [-k ]<br />
-H Server ip (default 127.0.0.1)<br />
-p Server port (default 5113)<br />
-c Number of parallel connections (default 5)<br />
-n Total number of requests (default 50)<br />
-k keep alive or reconnect (default is reconnect)</p>
<p align="left"><strong>执行结果</strong><br />
server:# ./benchmark -n 100<br />
5 parallel clients<br />
100 completed in 0 seconds<br />
1% &lt;= 6 milliseconds<br />
5% &lt;= 11 milliseconds<br />
15% &lt;= 12 milliseconds<br />
34% &lt;= 13 milliseconds<br />
89% &lt;= 17 milliseconds<br />
94% &lt;= 18 milliseconds<br />
99% &lt;= 21 milliseconds<br />
100% &lt;= 25 milliseconds<br />
177.936 requests per second</p>
<p align="left"><strong>脚本介绍</strong><br />
1）对分包（收&amp;发）进行了处理<br />
2）对超时进行了处理<br />
3）keepalive机制<br />
4）用户自定义并发数和请求数<br />
5）请求耗时百分比统计</p>
<p align="left"><strong>扩展性</strong><br />
考虑到扩展性，用户只需要重新定义下面两个函数即可。（组包和解包函数）<br />
int encodeRequest(char* data, unsigned &amp;len);<br />
int decodeResponse(char* data,unsigned len);</p>
<p align="left">ex：</p>
<ol start="1">
<li>int decodeResponse(char* data,unsigned len)</li>
<li>{</li>
<li>    char* ptr = data;</li>
<li>    int cmdLen = 0;</li>
<li>    if(*ptr != PRETX)</li>
<li>    {</li>
<li>        return -1;</li>
<li>    }</li>
<li>    else if(len &gt; 1)</li>
<li>    {</li>
<li>        cmdLen = ntohs(*(uint16_t*)(ptr + 1));</li>
<li>        if(cmdLen &lt;= 0 )</li>
<li>        {</li>
<li>            return -1;</li>
<li>        }</li>
<li>        else if(cmdLen &gt; (int)len)</li>
<li>        {</li>
<li>            //it is not a complete packet</li>
<li>            return 1;</li>
<li>        }</li>
<li>        else if( *(ptr + cmdLen -1) != LASTX)</li>
<li>        {</li>
<li>            return -1;</li>
<li>        }</li>
<li>    }</li>
<li>    return 0;</li>
<li>}</li>
</ol>
<p>&nbsp;</p>
