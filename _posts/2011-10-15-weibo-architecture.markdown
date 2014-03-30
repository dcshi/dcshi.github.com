---
layout: post
status: publish
published: true
title: "浅谈微博后台架构"
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 26
wordpress_url: http://www.dcshi.com/?p=26
date: '2011-10-15 10:35:39 +0000'
date_gmt: '2011-10-15 10:35:39 +0000'
categories:
- "架构"
tags:
- redis
- "架构"
---
<p>作为一个“局外人”，能拿到两家微博的技术资料少之又少，微博作为一个庞大的系统，我理解的也只是冰山一角<br />
（文中有些会带点主观思想，与事实不符敬请原谅）<br />
不谈论具体细节，只提供一个思考的方法。</p>
<p>sina微博资料：</p>
<p>http://www.slideshare.net/iso1600/high-performance-weibo-qcon-beijing-2011-7577912?from=ss_embed</p>
<p>http://www.slideshare.net/iso1600/ss-5861024?from=ss_embed</p>
<p>http://www.slideshare.net/iso1600/cache-4842490?from=ss_embed</p>
<p>OK，微博可以简单理解成一个feed系统或者邮件系统，你每发的一条消息就是一个群邮件，收件人就是你的粉丝。<br />
下面会提到的几个概念：<br />
扩散(我们内部叫扩散),分为三类，写扩散（push），读扩散（pull），还有混合扩散模式（push+pull）<br />
inbox：收件箱，你收到的消息（就是关注的人发的消息）<br />
outbox：发件箱，你发出的消息</p>
<p>写扩散：假设你的粉丝是1k,你每次发表一条微博，需要分别更新你的好友的inbox（inbox与outbox里只是一些ID list，实际数据只存一份，你懂的）<br />
缺点：<br />
写很重，每发表一次消息，导致大量的写操作，造成延时严重<br />
优点：<br />
读很轻，初始化首页只需要读取自己的inbox与outbox即可</p>
<p>读扩散：你每发表一条微博，只需要更新自己的inbox<br />
缺点：<br />
读很重，假设你收听了1K个好友，你每次刷新首页，你必须拿对比1K好友的推，然后计算得到最新的n条，这个是一个非常大的计算过程<br />
优点：<br />
写很轻，只需要更新自己的outbox。</p>
<p>不管是读扩散还是写扩散看上去都不是一个很好的解决方法。<br />
混合模式？<br />
就是push与pull结合使用。<br />
既然如果你的粉丝（这类人一般叫Idol）很多，你写扩散难度大，那你就不写扩散嘛。</p>
<p>我们来看下sina是怎样做的：<br />
下图是sina微博的cache架构图。</p>
<p><a href="http://www.dcshi.com/wp-content/uploads/2011/10/Image.png"><img class="alignnone size-full wp-image-27" title="Image" src="http://www.dcshi.com/wp-content/uploads/2011/10/Image.png" alt="" width="532" height="387" /></a></p>
<p>发表流程：（修改outbox-&gt;加载fans list-&gt;更新fans inbox）</p>
<p><a href="http://www.dcshi.com/wp-content/uploads/2011/10/Image1.png"><img class="alignnone size-full wp-image-28" title="Image" src="http://www.dcshi.com/wp-content/uploads/2011/10/Image1.png" alt="" width="728" height="488" /></a><br />
注解：假设你有1K的粉丝（我们假设少于2K粉丝的人不是Idol，我们采用写扩散），并不是你发的一条信息要更新1k次；这里sina做了个优化，<br />
只更新1k好友里面在线的人的inbox，为什么？你懂的</p>
<p>首页加载:(这里是指全量加载，轮询检查更新不走下面的流程)<br />
检查inbox是否可用（新登录而且还没有初始化过首页的用户是没有inbox）<br />
否 -&gt;加载following list -&gt;聚合消息（这里的计算量大）-&gt;update inbox<br />
是 -&gt;聚合inbox与outbox内容（由于Idoi不采用写扩散，那就是你的inbox不会有Idoi的更新数据，所以Idoi数据需要单独拉取并聚合）<br />
<img class="alignnone" title="pic" src="http://www.dcshi.com/wp-content/uploads/2011/10/Image2.png" alt="" width="749" height="545" /><br />
注解：上面很明显存在一个问题，当你首次登录的时候，你的inbox是空的，你需要初始化inbox是一个很大计算量的工作。假设你的关注人数是5K，<br />
你的首页需要展示最新的30条数据，那只能分别取出5K用户的最新的30条数据，并计算得到最新的30条。这里真的有必要去查询5K用户的消息吗？<br />
这里的优化方案跟inbox的更新策略一样，每个用户可以记录他最新的发表时间（如果用户登录后还没有发表过任何信息，那他的值是0），我们取出5K的好友id以及最新发表时间，做个简单的发表时间排序，拿出最新发表的30名用户ID，然后…..（你懂的）</p>
<p>到这里简单总结一下，每个在线的用户至少维护一个inbox，一个outbox，最新消息发表时间。此外，还有你的关注列表，粉丝列表，关注的Idol列表；从@TimyYang的ppt中不乏redis的身影，我猜想redis丰富的数据结构在sina微博的一些场景还是能占一席之位，而不只是做一些辅助的计算统计工作？，例如粉丝的set（ps，腾讯基本不采用这些开源的产品，但内部也不乏牛逼的产品）</p>
<p>OK，我们再看看最上面的cache架构图，还有一个叫content cache，他是什么用的？<br />
上面说到的inbox与outbox其实只是id list，实际的内容只存一份（不包含备份数据），content cache是对落地后的content的热点缓存。补充一点，我们看到content cache分为L0,L1,L2 这里我想是按时间区分的，具体策略可以参考：<a href="http://www.cnblogs.com/sunli/archive/2010/08/24/twitter_feeds_push_pull.html" target="_blank">http://www.cnblogs.com/sunli/archive/2010/08/24/twitter_feeds_push_pull.html</a>。<br />
对某一具体消息内容，sina微博采用mysql存储，腾讯微博采用SSD+大文件存储（每次写操作都是append操作，写操作可以先用内存缓存，达到适当大小合并，尽量减少随机写）。<br />
sina微博用的是innodb+innodb_buffer_pool+handlersocket的方法？具体的方案还真不是很清楚，但也没所谓了吧？<br />
腾讯微博的存储方案也相当的优雅：ssd随机读取性能高，用append的方式解决了随机写的问题。不用mysql，所以不用担心雪崩导致mysql撑不住了；具体的方案这里不讨论，有兴趣的同学可以私下聊</p>
<p>@TimYang的ppt还提到一个local cache的概念，这里的cache与前端机器是统一部署，是为了减少内网流量；想一下刘翔有1千万粉丝，每次拉取都造成内网流量，其实是不必要的，local cache很好解决了这个问题，但同时会增加逻辑复杂度。<br />
另外对于数据预热方面，@TimYang 提供了加锁的方式，防止并发加载同一个cache内容导致mysql垮掉。</p>
<p>&nbsp;</p>
