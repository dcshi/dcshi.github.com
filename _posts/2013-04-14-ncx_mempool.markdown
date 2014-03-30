---
layout: post
status: publish
published: true
title: ncx_mempool  轻量级内存池
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 360
wordpress_url: http://www.dcshi.com/?p=360
date: '2013-04-14 12:38:36 +0000'
date_gmt: '2013-04-14 12:38:36 +0000'
categories:
- server
- nginx
tags: []
---
<p>github地址: <a href="https://github.com/dcshi/ncx_mempool" target="_blank">https://github.com/dcshi/ncx_mempool</a>.</p>
<p>其中<a href="https://github.com/dcshi/ncx_mempool/blob/master/ncx_slab.c.orz" target="_blank">ncx_slab.c.orz</a>详细地进行了源码分析，所以文章不会对代码实现细节详细讨论。</p>
<p>&nbsp;</p>
<p>开发c、c++程序的时候，malloc、new等内存分配很函数很多时候会被使用到。<br />
但对于长期运行的后台服务来说，出于性能和内存碎片等考虑，我们的做法一般是重新实现内存池来管理服务的内存使用，而不是使用malloc、free等动态分配。</p>
<p>&nbsp;</p>
<p><a href="http://en.wikipedia.org/wiki/Slab_allocation" target="_blank">Slab Allocation</a>是一种很通用的内存池实现思路。如memcache，nginx，linux内核等都使用slab来做内存管理.<br />
它按照预先规定的大小，将分配的内存分割成特定长度的内存块，再把尺寸相同的内存块分成组，这些内存块不会释放，可以重复利用。<br />
至于实现细节上，却各有不同。</p>
<p>&nbsp;</p>
<ul>
<li>ncx_mempool 实质上是nginx的slab allcation实现。分离了nginx私有的部分逻辑，并添加了如查看内存池使用状态等功能而来。</li>
<li>ncx_mempool 非常轻量级、简单易用，详细可见<a href="https://github.com/dcshi/ncx_mempool/blob/master/README.md" target="_blank">github</a>的example和API使用说明.</li>
<li>ncx_mempool 性能比malloc明显要高，<a href="https://github.com/dcshi/ncx_mempool/blob/master/bench.c" target="_blank">bench.c</a> 分别测试了不同size的内存，各执行100w次分配和释放操作下，ncx_mempool 与 malloc的耗时。</li>
</ul>
<p><a href="http://www.dcshi.com/wp-content/uploads/2013/04/bench.png"><img class="alignleft size-full wp-image-367" title="bench" src="http://www.dcshi.com/wp-content/uploads/2013/04/bench.png" alt="" width="600" height="300" /></a></p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>ncx_mempool的源码分析请参考github的<a href="https://github.com/dcshi/ncx_mempool/blob/master/ncx_slab.c.orz" target="_blank">ngx_slab.c.orz</a>。接下来主要结合具体使用场景对内存结构进行分析。</p>
<p>&nbsp;</p>
<p>调用用例如下:</p>
<pre>ncx_slab_init(sp);
p1 = ncx_slab_alloc(sp, 5);
p2 = ncx_slab_alloc(sp, 1000);
p3 = ncx_slab_alloc(sp, 5000);
ncx_slab_free(sp, p1);
ncx_slab_free(sp, p2);
ncx_slab_free(sp, p3);</pre>
<p><strong>一.ncx_slab_init(sp) 后内存池结构:</strong><br />
<a href="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab1.png"><img class="alignleft size-full wp-image-386" title="ncx_slab1" src="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab1.png" alt="" width="1001" height="705" /></a><br />
<strong>二.ncx_slab_alloc(sp, 5) 后内存池结构:</strong><br />
<a href="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab2.png"><img class="alignleft size-full wp-image-387" title="ncx_slab2" src="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab2.png" alt="" width="1029" height="729" /></a></p>
<p><strong>三.ncx_slab_alloc(sp, 1000) 后内存池结构:</strong><br />
<a href="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab3.png"><img class="alignleft size-full wp-image-388" title="ncx_slab3" src="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab3.png" alt="" width="1029" height="729" /></a></p>
<p><strong>四.ncx_slab_alloc(sp, 5000) 后内存池结构:</strong><br />
<a href="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab4.png"><img class="alignleft size-full wp-image-389" title="ncx_slab4" src="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab4.png" alt="" width="1029" height="729" /></a></p>
<p><strong>五.依次对p1, p2, p3 释放以后内存结构:</strong><br />
<a href="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab5.png"><img class="alignleft size-full wp-image-390" title="ncx_slab5" src="http://www.dcshi.com/wp-content/uploads/2013/04/ncx_slab5.png" alt="" width="917" height="649" /></a></p>
