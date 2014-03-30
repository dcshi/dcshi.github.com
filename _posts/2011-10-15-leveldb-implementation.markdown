---
layout: post
status: publish
published: true
title: leveldb学习笔记 – implementation
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 19
wordpress_url: http://www.dcshi.com/?p=19
date: '2011-10-15 10:28:43 +0000'
date_gmt: '2011-10-15 10:28:43 +0000'
categories:
- server
- leveldb
tags:
- "笔记"
- leveldb
---
<p>leveldb的实现上借鉴了big table 思想（<a href="http://www.dcshi.com/?p=452" target="_blank">类bigtable数据存储模型| http://www.dcshi.com/?p=452</a>）</p>
<p>一.Log files<br />
*.log（如:000003.log ）顺序记录当前的更新的操作，每一个更新都是append操作。当log文件达到一定大小（默认是4M），它将会被重新存储为另一种log格式（SST sorted string table，每个sst的key都是有序的）<br />
大家都知道bigtable是Memtable+SStable的模型，leveldb的实现思想也是一样.*.log文件同时会在内存里保存一份（Memtable）</p>
<p>二.SST(Sorted String Table)<br />
SST存储在磁盘，而且SST的记录是根据key排好序的.<br />
SST中的每一条记录都是K-V的形式，K是key；value有两种情况，其一是真实数据，其二是删除标记<br />
SST分为多个级别（level，这也是leveldb的名字由来），从log文件生成的是level-0.然而leve-0的SST数目不能超过一个阀值（默认是4），如果超过阀值，后台线程会对所有的level-0的SST和存在key重叠的level-1 的SST进行合并处理，生成新的level-1的SST<br />
在level-0中的每个SST之间可能存在重复的key，但在其他level内的SST之间是保证不存在有重叠的区间</p>
<p>三.Manifest<br />
Manifest记录了每个level下的有哪些SST，相应的key取值区间（key是排序的），和一些重要的元数据<br />
每次database被重新打开,都会重新生成一个Manifest文件<br />
Manifest也正如一个日志文件一样，任何服务状态的改变，都会在Manifest 追加一条记录</p>
<p>四.Current<br />
Current的用途很简单，记录当前最新的Manifest的名字</p>
<p>五.Info logs<br />
Informational messages are printed to files named LOG and LOG.old.(注意这里的log跟上面提到的*.log是不同的)</p>
<p>六.对level-0的简单描述：<br />
当*.log文件大小超过一定的阀值（默认是1M）：<br />
1.创建一个新的log文件和Memtable，后续的更新操作都会作用在这<br />
2.后台操作：<br />
1）把旧的Memtable的数据写到一个新的SST中<br />
2）删除旧的Memtable<br />
3）删除旧的log文件<br />
4）把刚才新生成的SST添加到level-0去</p>
<p>七.Compaction：<br />
Compaction的条件：<br />
（1）对于level 0，如果总的disk table个数超过一定阈值（默认是4）<br />
（2）对于其他level L， 如果disk table的总大小超过一定阈值（对于level 1是10M，level 2是100M，依此类推）<br />
（3）如果某个disk table被读取的次数超过了一定阈值（文件大小/16K），也会对该disk table进行compaction。（当一次Get操作有多个disk table被读取的时候，只有第一个被统计了次数）</p>
<p>Compaction的处理：<br />
（1）level-L某个SST做Compaction的时候需要检查level-（L+1）的SST中是否有重叠的key；如果存在重叠的，则把level-L的SST与Level-（L+1）的SST进行Compaction，生成新的Level-（L+1）SST，并把原来旧的SST丢弃<br />
（2）正如上面说的由于level0中的每个SST之间可能存在重叠的key，所以level-0做Compaction的时候是跟其他level有点区别。level0中SST之间就可能存在重叠的部分，所以Level-0做Compaction的时候会在Level-0中取多个SST与Level-1中出现重叠key的SST做Compaction操作</p>
<p>Compaction操作是根据key来的；每个Level做完一次Compaction操作，系统会记录该Level最后一次Compaction操作的ending key，所以下次Compaction操作会从改key的下一个key开始</p>
<p>Compaction除了删除重复的key，还会对一些标有delete marker的key进行清理</p>
<p>八.Timing<br />
对Level-0（每个SST文件是1M） Compaction操作，最坏的情况是4*1+10=14M<br />
除此之外，其他Level 进行Compaction操作也会大量的IO</p>
<p>思考一个问题：<br />
如果频繁的写操作，导致Level-0的文件剧增.。如果频繁Compaction，会导致大量的IO，严重影响到系统的性能；如果不Compaction，每次读的需要查找多个SST，代价也是巨大的，该如何来解决？</p>
<p>九.Recovery<br />
1)从CURRENT找到当前使用的Manifest<br />
2)从Manifest读取信息<br />
3)清理没有用的文件<br />
4)是否需要打开所有的SST，官方的做法是lazy loading<br />
5)Convert log chunk to a new level-0 sstable<br />
6)Start directing new writes to a new log file with recovered sequence</p>
<p>十.写性能分析<br />
每次写操作只涉及到一次对Log文件的顺序写操作，并且默认模式下只是只是写到系统缓存，再由系统跟硬盘进行同步，所以写操作的效率非常高。</p>
<p>十一.读性能分析<br />
根据Compaction条件的第（1）和第（2）条，假设数据库大小为n MB，则最坏情况下需要4+log10(n)次读取Disk Table的操作。不过，根据第（3）条，多次被访问的Disk Table也会被Compaction，因此多次访问后均摊下来的读盘次数其实是非常小的。</p>
