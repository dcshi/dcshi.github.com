---
layout: post
status: publish
published: true
title: "无状态UUID server设计"
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 10
wordpress_url: http://www.dcshi.com/?p=10
date: '2011-10-15 10:24:09 +0000'
date_gmt: '2011-10-15 10:24:09 +0000'
categories:
- server
tags:
- "性能"
---
<p>作为一个全局唯一的UUID server，需要保证一下几个维度</p>
<p>1）服务可支持秒级重启<br />
2）对机器时间回滚有容错<br />
3）支持分布式部署<br />
4）支持单机部署多个UUID 服务</p>
<p>上面四个最基本的需求就要求我们设计的UUID server是没有状态，可以任意重启。<br />
先想想我们一般是怎么来做UUID的？<br />
一。之前看到过很多朋友利用mysql的auto_increment特性来做uuid。<br />
1）性能，互联网应用性能瓶颈估计很多人会提到db，那你还用mysql做UUID server吗？<br />
2）假设需要部署多个uuid做负载均衡，mysql 的auto_increment能做到吗？<br />
3）假设磁盘坏了（虽然概率低，但不能排除），那不是悲剧了？<br />
问题还是各种各样的….</p>
<p>二。linux系统提供了实现，具体可以参考<a href="http://linux.die.net/man/3/uuid_generate" target="_blank">http://linux.die.net/man/3/uuid_generate</a><br />
uuid_generate系列函数用了随机算法产生UUID，实现不透明，也很难保证上面提到的四个基本需求。</p>
<p>实现UUID的方法有很多很多，以下是个人的想法：<br />
+++++++++++++++++++++++++++++++++<br />
| 进程id | cur_time | counter |<br />
+++++++++++++++++++++++++++++++++<br />
我想通过三个维度来实现全局唯一：<br />
1）进程id（不管在分部署式系统或者是单机上部署多服务，都需要保证进程的id是唯一的，至于如何保证进程id唯一性，稍后讨论）<br />
2）当前时间（用time_t表示即可，它是一个有符号整形）<br />
3）自增id。</p>
<p>首先，如何设计进程id。我首先要问自己一个问题，你的uuid server规模是多大？少于10台（考虑到UUID server的工作简单，单机性能也会很高，所以我觉得需要10台以上UUID server的应用规模已经是相当大），这样我建议你可以通过配置文件（作为一个server，不可能没有配置文件吧？）的一个字段来表明，简单即美。如果你的规模真的那么大？如果动态维护不同的进程标识？请参考<a href="http://weibo.com/n/bnu_chenshuo" target="_blank">陈硕老师</a>的《<a href="http://blog.csdn.net/solstice/article/details/6285216" target="_blank">分布式系统中的进程标识</a>》，以四元组 ip:port:start_time:pid 作为分布式系统中进程的 gpid</p>
<p>其次，当前时间。这个很好理解了，直接调用time(NULL)返回当前系统的秒数</p>
<p>最后，自增id，这里称作counter。这个也是比较关键。四个基本需求里，提到可以支持秒级重启。如果counter是uuid server的一个全局变量，每次重启都从0开始，假设一秒内的并发很高，那就会出现有重复id生成的情况了。所以counter不能作为uuid server的一个全局变量。所以我们把它放到机器的共享内存里。这样尽管进程随便重启，counter不会从0开始。那什么情况counter又0开始呢？机器重启，我们相信机器重启不会在秒级完成，一旦不是在同一秒内重启，那样我们上面提到的time已经发生变化，counter尽管跟之前重复，也不会造成uuid重复了。</p>
<p>好了，基于这个方案，回顾下上面提到的四个基本需求是否得到满足<br />
1）秒级重启，这个不是问题<br />
2）时间回滚有容错。如果当前时间是1314000000，运行了10秒以后，机器时间按变成1314000010，由于时间同步的作用，server当前时间又变成1314000000，但是我们还有counter，由于counter不会回滚，所以对时间回滚也有容错<br />
3)支持分布式部署，这得益于进程id的唯一性，可能有同学会想，用IP,或者MAC地址也可以保证唯一性。那让我们再看第四个需求<br />
4）支持单机部署多个UUID 服务,由于进程id的唯一性，所以单机部署多个uuid服务是没有问题（这里就体现到进程id的相比机器IP,MAC地址有优势了,IP/MAC只能保证机器唯一，不能保证进程唯一）</p>
<p>&nbsp;</p>
