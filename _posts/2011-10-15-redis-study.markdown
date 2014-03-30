---
layout: post
status: publish
published: true
title: redis学习笔记：从acceptCommonHandler看redis commands解析
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 17
wordpress_url: http://www.dcshi.com/?p=17
date: '2011-10-15 10:27:55 +0000'
date_gmt: '2011-10-15 10:27:55 +0000'
categories:
- server
- redis
tags:
- redis
- "笔记"
---
<p>今天的学习笔记，有点乱，重在理解过程。</p>
<p>redis在接收到client的连接时，首先会创建一个redisClient。<br />
redisClient创建成功以后，判断maxclients是否超出，如果超出最大连接数，redis会给客户端发送errors：-ERR max number of clients reached\r\n,并把该clinet free掉.</p>
<p>- createClient<br />
创建一个redisClient，并为accpet的fd创建一个fileEvent(aeCreateFileEvent).并注册readQueryFromClient事件</p>
<p>-readQueryFromClient<br />
接收网络流，保存到redisClient的querybuf中，并调用processInputBuffer(c)做相应的逻辑处理</p>
<p>– processInputBuffer<br />
首先根据querybuf[0] == ‘*’判断是REDIS_REQ_MULTIBULK or REDIS_REQ_INLINE</p>
<p>– processInlineBuffer<br />
了解inline commands的协议，这里主要做协议解析，把相应的参数放到redisClinet的argv（robj**）<br />
协议见附(a)</p>
<p>– REDIS_REQ_MULTIBULK<br />
了解multibulk协议，这里同样是一个协议解析过程，把相应的参数放到redisClinet的argv（robj**）<br />
协议见附(a)</p>
<p>– processCommand(c)处理请求命令的逻辑，处理完调用resetClient初始化redisClient<br />
通过lookupCommand把commandname转换成对应的函数指针<br />
接着是一系列的逻辑判断：<br />
1）判断该redisClient是否sub/pub<br />
2）判断该sever是否slave，并且检查slave-serve-stale-data is yes or no（slave是否已从master同步完数据）<br />
3）server是否处于loading状态（例如server刚重启，需要从快照或者aof中loading数据，这个时候不能提供正常服务）<br />
4)判断 client is in a MULTI context or not（例如pipeline）<br />
以上的种种情况，redis只能允许执行少部分命令。例如4）如果clinet is in a MULI context，则如果命令不是execCommand 等，则需要调用queueMultiCommand把当前的命令放到redisClient中的mstate（实质上是一个command list，具体看multiState这个数据结构）</p>
<p>OK，命令的执行通过call函数，他接受两个参数：redisClient和redisCommand 。<br />
在讨论call函数之前，我们看到这样一段代码<br />
if (server.vm_enabled &amp;&amp; server.vm_max_threads &gt; 0 &amp;&amp; blockClientOnSwappedKeys(c,cmd))<br />
return REDIS_ERR;</p>
<p>介绍一下blockClientOnSwappedKeys(redisClient *c, struct redisCommand *cmd) {：<br />
1）判断是cmd是否有vm_preload_proc（Use a function to determine which keys need to be loaded<br />
in the background prior to executing this command.）函数，如果存在则先执行。其实vm_preload_proc的用意很明显，有些函数执行需要一些前提条件，这样通过vm_preload_proc来提供，例如exec的vm_preload_proc函数就是一次执行redisClient-&gt;mstate中的函数。<br />
2）waitForMultipleSwappedKeys (redisClient *c, struct redisCommand *cmd, int argc, robj **argv)，从名字上就你猜出，该函数的作用是把需要的值从虚拟内存中load到内存的。<br />
核心在waitForSwappedKey：<br />
a)通过dictFind找到key对应的val（redis中的key-val都是一个dictEntry的结构体),val是一个robj结构，通过robj-&gt;storage判断val存储状态（ REDIS_VM_MEMORY or REDIS_VM_SWAPPING，REDIS_VM_SWAPPED，REDIS_IOJOB_LOAD，分表表示在内存，正在swapping到disk，在disk，正在从disk加载到内存）。如果数据不在内存，需要从disk中加载，并需要把此redisClient 阻塞掉。<br />
b)redis是怎么处理阻塞操作以保证数据ready后能正常唤醒被阻塞的redisClient？redis做了如下操作：<br />
[1].把阻塞的key添加到c-&gt;io_keys（just a list）<br />
[2].把阻塞的key添加到c-&gt;db-&gt;io_keys(显示一个dictFind，如果操作不到才会添加该key)<br />
[3].Add the client to the swapped keys =&gt; clients waiting map。首先你必须要知道c-&gt;db-&gt;io_keys是一个dict结构（简单理解成hashtable，冲突用list解决）。hashtable中的的key-val也是一个dictEntry结构，key是被阻塞的key，val是被阻塞的redisClient，这步主要是建立了一个key=&gt;client的mapping.<br />
[4].那阻塞的client什么时候被唤醒？在aeMain中，每次执行循环之前会判断beforesleep是否存在，beforesleep主要完成两个事情：处理完成IO loading的clients（遍历server.io_ready_clients）和处理阻塞的clients（遍历server.unblocked_clients）（例如BLPOP等）。<br />
c)如果需要的数据是REDIS_VM_SWAPPED状态，即在disk上，需要执行queueIOJob创建一个IO task，执行步骤如下：<br />
[1].往server.io_newjobs中添加一个iojob（包含db，key，所需要的page，所属的thread等，这里补充下，redis的swap操作是多线程模型）<br />
[2].判断如果io_active_threads还没有达到上限，则创建一个io线程。io线程的主要逻辑在IOThreadEntryPoint：取出server.io_newjob中的一个job（由于是多线程，所以这里可以见到redis中少有的锁操作）；把job放到server.io_processing list中；如果job处理完，删除server.io_processeing中的job，添加到server.io_processed去。在函数的最后我们看到一个write(server.io_ready_pipe_write,”x”,1)操作，io线程与主线程之间通过pipe（管道）通信，这里是告诉main thread，一个io操作完成了（For every byte we read in the read side of the pipe, there is one I/O job completed to process）。当然一个io完成以后，redis要做大量的事情，其中一个就是handleClientsBlockedOnSwappedKey，即上面提到的逆过程：1）在c-&gt;io_keys中删除该key 2）删除c-&gt;db-&gt;io_keys中key下该redisClient节点 3)把完成io等待的redisClient添加到server.io_ready_clients中。<br />
3）If the client was blocked for at least one key, mark it as blocked.。并调用aeDeleteFileEvent删除该redisClient的可读事件。</p>
<p>好，一对废话后回到call函数（感觉我的思路就像解析器解析程序一样，不断的有函数出栈入栈，-_-”）。<br />
void call(redisClient *c, struct redisCommand *cmd) {<br />
long long dirty;<br />
/*<br />
下面一轮减法的作用是为了得到dirty的值有没有改变，以判断是否需要写aof（如果数据只读没写，即没有dirty，所以也不需要记录在aof中去）<br />
*/<br />
dirty = server.dirty;<br />
cmd-&gt;proc(c); //调用具体函数<br />
dirty = server.dirty-dirty;<br />
if (server.appendonly &amp;&amp; dirty)<br />
feedAppendOnlyFile(cmd,c-&gt;db-&gt;id,c-&gt;argv,c-&gt;argc); //就是把操作记录在aof<br />
if ((dirty || cmd-&gt;flags &amp; REDIS_CMD_FORCE_REPLICATION) &amp;&amp;<br />
listLength(server.slaves))<br />
replicationFeedSlaves(server.slaves,c-&gt;db-&gt;id,c-&gt;argv,c-&gt;argc); //同步到slave<br />
if (listLength(server.monitors))<br />
replicationFeedMonitors(server.monitors,c-&gt;db-&gt;id,c-&gt;argv,c-&gt;argc); //顾名思义同步信息给Monitor<br />
server.stat_numcommands++;<br />
}</p>
<p>附(a)redis协议见：http://redis.io/topics/protocol<br />
client发布命令的格式有如下几种，第一个字符串都必须是命令字，不同的参数之间用1个空格来分隔：<br />
1）Inline Command:：仅一行<br />
比如 EXISTS命令，client发送的字节流类似于”EXISTS mykey\r\n”。<br />
2）Bulk Command：类似于返回协议的bulk reply，一般有两行，第一行依次为“命令字 参数 一个数字”，该数字表示下一行字符的个数<br />
比如SET命令，client发送的字节流类似于”SET mykey 5\r\nhello\r\n”。<br />
3）multi-bulk Command：跟返回协议的multi-bulk reply类似。<br />
比如上面的SET命令，用multi-bulk协议表示则为“*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$5\r\nhello\r\n”。<br />
尽管对于某些命令该协议发送的字节流比bulk command形式要多，但它可以支持任何一种命令，支持跟多个二进制安全参数的命令（bulk command仅支持一个），也可以使得client library不修改代码就能支持redis新发布的命令（只要把不支持的命令按multi-bulk形式发布即可）。redis的官方文档中还提到，未来可能仅支持client采用multi-bulk Command格式发布命令。<br />
另外提一下，client library可以连续发布多条命令，而不是等到redis返回前一条命令的执行结果才发布新的命令，这种机制被称作pipelining，支持redis的client library大多支持这种机制，读者可自行参考。</p>
<p>最后来看看redis实现时用来返回信息的相关函数。<br />
redis会使用addReplySds、addReplyDouble、addReplyLongLong、addReplyUlong、addReplyBulkLen、addReplyBulk、addReplyBulkCString等来打包不同的返回信息，最终调用addReply来发送信息。<br />
addReply会将发送信息添加到相应redisClient的reply链表尾部，并使用sendReplyToClient来发送。sendReplyToClient会遍历reply链表，并依次发送，其间如果可以打包reply（server.glueoutputbuf为真），则可以使用glueReplyBuffersIfNeeded把reply链表中的值合并到一个缓冲区，然后一次性发送。</p>
<p>&nbsp;</p>
