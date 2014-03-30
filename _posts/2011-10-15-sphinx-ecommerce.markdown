---
layout: post
status: publish
published: true
title: Sphinx在电子商务网站中的应用
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 33
wordpress_url: http://www.dcshi.com/?p=33
date: '2011-10-15 10:37:59 +0000'
date_gmt: '2011-10-15 10:37:59 +0000'
categories:
- sphinx
tags:
- sphinx
---
<p align="left">转自 <a href="http://www.hihiyou.com/?cat=4" target="_blank">http://www.hihiyou.com/?cat=4</a></p>
<p align="left"><strong>第一部分</strong><strong>-</strong><strong>分析与思路：</strong><br />
实际项目的特点是：<br />
1.数据量大<br />
2.商品分类多<br />
3.每个商品可以处于多个分类下<br />
4.平均每个商品拥有2个以上的属性</p>
<p align="left">要求可以做到：<br />
1.按照商品名称搜索<br />
2.按照商品属性筛选<br />
3.按照分类筛选<br />
4.按照价格分组筛选<br />
5.按照价格、时间排序</p>
<p align="left">MySQL基本数据结构：</p>
<p align="left"><a href="http://www.dcshi.com/wp-content/uploads/2011/10/未命名.jpg"><img class="alignnone size-full wp-image-36" title="未命名" src="http://www.dcshi.com/wp-content/uploads/2011/10/未命名.jpg" alt="" width="608" height="142" /></a></p>
<p align="left">说明:<br />
product是基本产品表，id为产品编号，name是待搜索字段<br />
product_index为产品属性表，index_id是属性编号，product_id为product表的id。如果编号为100的产品，有5个不同属性，在product_index表中就会记录product_id为100，index_id不相同的五条记录。<br />
product_show的结构与product_index类似，记录的是产品于分类的联系，如果同一个产品同属5个不同的分类，在表中就会存在五条记录。</p>
<p align="left">数据结构不复杂，但是实际应用中，使用Mysql的查询实现的搜索和检索，效率是比较低的。最消耗性能的地方是产品列表的索引导航，这是类似淘宝网产品列表的”按XXX浏览”：</p>
<p align="left">SQL的实现不多说。</p>
<p align="left">在Sphinx有个非常棒的特性：MVA(多值属性），MVA类似Mysql的set类型的字段，但是存放值不受预设的限制，使用MVA建立Sphinx的属性，用来存放每个产品的index_id和category_id，很方便就能实现筛选功能。</p>
<p align="left"><strong>第二部分</strong><strong>-</strong><strong>索引结构配置：</strong><br />
Sphinx配置文件<br />
贴出主要部分，其他的手册上说的很清楚了（手册非常重要：<a href="http://www.wapm.cn/uploads/pdf/sphinx_doc_zhcn_0.9.pdf" target="_blank">http://www.wapm.cn/uploads/pdf/sphinx_doc_zhcn_0.9.pdf</a>）。</p>
<p align="left"><a href="http://www.dcshi.com/wp-content/uploads/2011/10/未命名1.jpg"><img class="alignnone size-full wp-image-37" title="未命名" src="http://www.dcshi.com/wp-content/uploads/2011/10/未命名1.jpg" alt="" width="608" height="293" /></a></p>
<p align="left">上面是source部分，项目的编码是UTF-8，所以首先得执行：“SET NAMES utf8”。product里面的price是以保留两位小数的形式存放的，在sql_query里面，将它乘以100换成整数，避免浮点精度带来的不必要的小数位。<br />
关键的两条“sql_attr_multi”，配置了属性和分类的MVA，“from query”后面紧接着SQL，表示来自查询，product_id放第一个字段，后面跟着值字段，这样Sphinx内部才会进行处理。</p>
<p align="left">建立索引后,结构可以看成下图所示:</p>
<p align="left"><a href="http://www.dcshi.com/wp-content/uploads/2011/10/index.gif"><img class="alignnone size-full wp-image-38" title="index" src="http://www.dcshi.com/wp-content/uploads/2011/10/index.gif" alt="" /></a></p>
<p align="left"><strong>第三部分</strong><strong>-</strong><strong>查询：</strong><br />
Sphinx自带的PHP的API进行查询。<br />
示例1：<br />
编号为1,2,3,4,5的5个分类下的产品列表：</p>
<ol start="1">
<li>$Sphinx-&gt;SetFilter( 'category_id', array(1,2,3,4,5) );</li>
<li>$Sphinx-&gt;Query( '', '*' );</li>
</ol>
<p align="left">示例2：<br />
属性编号为1,2,3,4,5的下的产品列表(属性的交集)</p>
<ol start="1">
<li>$Sphinx-&gt;SetFilter( 'index_id', array(1) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(2) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(3) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(4) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(5) );</li>
<li>$Sphinx-&gt;Query( '', '*' );</li>
</ol>
<p align="left">示例3：<br />
属性编号为1,2,3,4,5的下的产品列表(属性的交集)的价格分组,并且升序排列</p>
<ol start="1">
<li>$Sphinx-&gt;SetFilter( 'index_id', array(1) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(2) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(3) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(4) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(5) );</li>
<li>$Sphinx-&gt;SetGroupBy( 'price', SPH_GROUPBY_ATTR, '@group asc' );</li>
<li>$Sphinx-&gt;Query( '', '*' );</li>
</ol>
<p align="left">示例4：<br />
属性编号为1,2,3,4,5的下的产品列表(属性的交集),分类编号是7,8,9</p>
<ol start="1">
<li>$Sphinx-&gt;SetFilter( 'category_id', array(7,8,9) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(1) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(2) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(3) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(4) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(5) );</li>
<li>$Sphinx-&gt;SetGroupBy( 'price', SPH_GROUPBY_ATTR, '@group asc' );</li>
<li>$Sphinx-&gt;Query( '', '*' );</li>
</ol>
<p align="left">Sphinx的查询使用还是很方便的,需要注意的是,Filter传递的待筛选参数必须是数组;</p>
<p align="left">关于MVA的使用,如果数组有多个成员:</p>
<ol start="1">
<li>$Sphinx-&gt;SetFilter( 'category_id', array(7,8,9) );</li>
</ol>
<p align="left">表示索引中的category_id满足数组中的任意一个值就会筛选出来.</p>
<ol start="1">
<li>$Sphinx-&gt;SetFilter( 'index_id', array(7) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(8) );</li>
<li>$Sphinx-&gt;SetFilter( 'index_id', array(9) );</li>
</ol>
<p align="left">表示索引中的index_id包含7,8,9三个值才会被筛选出.</p>
<p align="left">MVA也可以用来进行Group操作:<br />
$Sphinx-&gt;SetGroupBy( 'index_value_id', SPH_GROUPBY_ATTR );</p>
<p align="left">可以很方便的得出当前条件下的,各种属性(样式,品牌…)的数量.</p>
<p align="left"><strong>第四部分</strong><strong>-</strong><strong>即时更新方案：</strong><br />
使用了Sphinx代替了Mysql进行查询后,最大的问题还在于数据的更新问题:我们在后台进行了产品的编辑,删除,添加操作,都需要尽快的反应到用户端.<br />
架构图:</p>
<p align="left"><a href="http://www.dcshi.com/wp-content/uploads/2011/10/index-update.gif"><img class="alignnone size-full wp-image-39" title="index-update" src="http://www.dcshi.com/wp-content/uploads/2011/10/index-update.gif" alt="" /></a></p>
<p align="left">说明:<br />
(1).在Mysql中建立一个增量表,凡是产品更新操作(添加,修改,删除),都会将相关ID更新到到增量表;表结构很简单,只记录ID.</p>
<p align="left">(2).修改Sphinx的配置文件</p>
<ol start="1">
<li>sql_query = SELECT id, name, price*100 AS price, 0 AS in_update FROM shop_product</li>
<li>sql_attr_uint = in_update</li>
</ol>
<p align="left">增加了一个字段in_update,用来标记主索引的这条记录,是否在增量表中.</p>
<p align="left">(3).产品的更新操作时,除了更新增量表,同时也利用Sphinx API的UpdateAttributes方法,将主索引中的相关记录的in_update属性设置成”1″</p>
<ol start="1">
<li>$this-&gt;Sphinx-&gt;UpdateAttributes( $index, array( 'in_update' ), array( $id =&gt; '1' ) );</li>
</ol>
<p align="left">代码中的$id指的是商品编号.</p>
<p align="left">更新完了属性后,发出增量索引更新通知,可以是写队列,写文件等方式.</p>
<p align="left">(4).在查询时,增加过滤器.</p>
<ol start="1">
<li>$Sphinx-&gt;SetFilter( 'in_update', array(0) );</li>
</ol>
<p align="left">这样,就不会使用到主索引中的处于增量索引中的doc,以免搜索到错误的编辑前的数据.</p>
<p align="left">(5).守护进程接到增量索引更新通知,重建增量索引.</p>
<p align="left">(6).每天某时段更新主索引,清空增量表,清空增量索引…</p>
<p align="left">思路就是这样,下面的增量索引更新脚本是我用在测试环境的,生产环境的比较复杂,供参考.</p>
<ol start="1">
<li>#!/bin/bash</li>
<li>CHECK_FILE_PATH='/xxx/sphinx-check.txt'</li>
<li>SPHINX_COMMAND="/opt/sphinx/bin/indexer –config /opt/sphinx/etc/product.conf product_update_index –rotate"</li>
<li>while true</li>
<li>do</li>
<li> NOW_TIME=`date +'%Y-%m-%d %H:%M:%S'`</li>
<li> if [ -f $CHECK_FILE_PATH ];then</li>
<li>  WORD_NUM=`cat ${CHECK_FILE_PATH} |wc -w`</li>
<li>  if [ $WORD_NUM -gt 0 ];then</li>
<li>   echo "bulid    ${NOW_TIME}"</li>
<li>   $SPHINX_COMMAND</li>
<li>   &gt; $CHECK_FILE_PATH</li>
<li>  else</li>
<li>   echo "skip    ${NOW_TIME}"</li>
<li>  fi</li>
<li> else</li>
<li>  echo "nofile    ${NOW_TIME}"</li>
<li> fi</li>
<li> sleep 30</li>
<li>done</li>
</ol>
<p align="left">后台操作时,简单的用写文件的方式通知索引更新脚本,索引更新完成后,将文件重置.</p>
<p align="left"><strong>第五部分</strong><strong>-</strong><strong>优化总结：</strong><br />
这几篇blog,没想过写成手把手的教程,只是想把思路写出来,和大家交流,所以流水账了…代码都是些关键片段,不负责它正常运行!<br />
基础方面的请到Sphinx官方站查看文档,非常详细!</p>
<p align="left">谈几点优化方面的建议:</p>
<p align="left">1.Sphinx自带PHP API和使用C写的PHP扩展相比,速度快了不少,应该是网络通讯方面的问题.觉得这应该是不正常的,但是问题我还没找到.也有可能是个别问题.</p>
<p align="left">2.一个页面要是有多个Sphinx查询请求,请使用AddQuery方法,尽量收集,一次发送,一个接收.</p>
<p align="left">3.AddQuery添加Query的条数有限制,应该是32条这样,我以前提到过.</p>
<p align="left">4.做好cache!</p>
<p>&nbsp;</p>
