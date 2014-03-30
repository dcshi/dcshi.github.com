---
layout: post
status: publish
published: true
title: ngx_lua 学习笔记 - overview
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "<h3>what ngx_lua</h3>\r\n<a href=\"http://openresty.org/\" target=\"_blank\">openresty</a>
  与 <a href=\"http://wiki.nginx.org/HttpLuaModule\" target=\"_blank\">ngx_lua</a>。
  很多时候，我们会把这两个技术拉在一起讨论，他们的关系：\r\n<ul>\r\n\t<li>同一个父亲，主要维护者<a href=\"http://agentzh.org\"
  target=\"_blank\">agentzh</a></li>\r\n\t<li>ngx_lua 是 nginx的一个模块；openresty是nginx+nginx
  常用模块的一个集合，即openretsy包括了ngx_lua.</li>\r\n</ul>\r\n<p style=\"text-indent: 1em;\">简单总结，openresty与ngx_lua没有必然的依赖关系。使用openresty，不一定要用到ngx_lua；同样，我们可以单独下载ngx_lua，编译使用。</p>\r\n&nbsp;\r\n<h3>why
  ngx_lua</h3>\r\n<ul>\r\n\t<li><span style=\"text-indent: 1em;\">nodejs目前是最火的技术之一，基于google
  V8 引擎 + javascript天生支持callback异步回调机制，使得js在后端运行成为了可能。了解过nodejs的人都会被它的简单和性能所惊讶。使用过nodejs的人会被大量的callback嵌套所厌烦.（当然有<a
  href=\"http://windjs.org/cn/\" target=\"_blank\">windjs</a>等解决方案）。</span></li>\r\n\t<li><span
  style=\"text-indent: 1em;\">ngx_lua基于nginx + lua，前者的以性能高著称，后者原生支持<a href=\"http://en.wikipedia.org/wiki/Coroutine\"
  target=\"_blank\">协程</a>，满足非阻塞模型的基础条件，而且lua语言设计上的简单，使得lua也被誉为性能最高的脚本语言之一(特别是运行在<a
  href=\"http://luajit.org/\" target=\"_blank\">luajit</a>之上)。ngx_lua相比nodejs，对cpu的占用更少(同等的场景下)，使得ngx_lua性能比nodejs高成为可能；ngx_lua原生支持用同步语法写异步代码。</span></li>\r\n</ul>\r\n<p
  style=\"text-indent: 1em;\">简单的总结，我们可以基于ngx_lua来完成传统cgi、fastcgi的大部分工作；至于是node还是ngx_lua，这需要你对两个技术充分了解的前提下，并结合业务需要，做出权衡。值得一题的是，某个技术的用与不用，可控性是不得不考虑的因素，盲目追求性能是没有意义的。</p>\r\n&nbsp;\r\n<h3>how
  ngx_lua</h3>\r\n<ul>\r\n\t<li> 每Nginx工作进程使用一个Lua VM，工作进程内所有协程共享VM</li>\r\n\t<li>
  将Nginx I/O原语封装(即nginx的API，如时间函数等)后注入Lua VM，允许Lua代码直接访问</li>\r\n\t<li> 每个外部请求都由一个Lua协程处理，协程之间数据隔离</li>\r\n\t<li>
  Lua代码调用I/O操作接口时，若该操作无法立刻完成，则打断相关协程的运行并保护上下文数据，I/O操作完成时还原相关协程上下文数据并继续运行</li>\r\n</ul>\r\n<p
  style=\"text-indent: 1em;\">简单总结，ngx_lua只是nginx的一个工作模块，它并不是服务；why lua，not php..?
  nginx在服务过程中，并不会随着请求数的递增，主动派生子进程、子线程。nginx之所以基于固定的几个工作进程即可达到高性能，是因为基于非阻塞的前提，即任何请求不能对nginx造成阻塞。lua原生支持协程机制，使得业务出现阻塞（如等待mysql返回）可以主动放弃执行权，并让nginx继续执行其他请求；直到原来请求执行条件满足（如mysql返回数据），lua可以在原来执行到的地方重新执行。</p>\r\n"
wordpress_id: 317
wordpress_url: http://www.dcshi.com/?p=317
date: '2013-03-16 08:58:55 +0000'
date_gmt: '2013-03-16 08:58:55 +0000'
categories:
- nginx
- ngx_lua
tags: []
---
<h3>what ngx_lua</h3>
<p><a href="http://openresty.org/" target="_blank">openresty</a> 与 <a href="http://wiki.nginx.org/HttpLuaModule" target="_blank">ngx_lua</a>。 很多时候，我们会把这两个技术拉在一起讨论，他们的关系：</p>
<ul>
<li>同一个父亲，主要维护者<a href="http://agentzh.org" target="_blank">agentzh</a></li>
<li>ngx_lua 是 nginx的一个模块；openresty是nginx+nginx 常用模块的一个集合，即openretsy包括了ngx_lua.</li>
</ul>
<p style="text-indent: 1em;">简单总结，openresty与ngx_lua没有必然的依赖关系。使用openresty，不一定要用到ngx_lua；同样，我们可以单独下载ngx_lua，编译使用。</p>
<p>&nbsp;</p>
<h3>why ngx_lua</h3>
<ul>
<li><span style="text-indent: 1em;">nodejs目前是最火的技术之一，基于google V8 引擎 + javascript天生支持callback异步回调机制，使得js在后端运行成为了可能。了解过nodejs的人都会被它的简单和性能所惊讶。使用过nodejs的人会被大量的callback嵌套所厌烦.（当然有<a href="http://windjs.org/cn/" target="_blank">windjs</a>等解决方案）。</span></li>
<li><span style="text-indent: 1em;">ngx_lua基于nginx + lua，前者的以性能高著称，后者原生支持<a href="http://en.wikipedia.org/wiki/Coroutine" target="_blank">协程</a>，满足非阻塞模型的基础条件，而且lua语言设计上的简单，使得lua也被誉为性能最高的脚本语言之一(特别是运行在<a href="http://luajit.org/" target="_blank">luajit</a>之上)。ngx_lua相比nodejs，对cpu的占用更少(同等的场景下)，使得ngx_lua性能比nodejs高成为可能；ngx_lua原生支持用同步语法写异步代码。</span></li>
</ul>
<p style="text-indent: 1em;">简单的总结，我们可以基于ngx_lua来完成传统cgi、fastcgi的大部分工作；至于是node还是ngx_lua，这需要你对两个技术充分了解的前提下，并结合业务需要，做出权衡。值得一题的是，某个技术的用与不用，可控性是不得不考虑的因素，盲目追求性能是没有意义的。</p>
<p>&nbsp;</p>
<h3>how ngx_lua</h3>
<ul>
<li> 每Nginx工作进程使用一个Lua VM，工作进程内所有协程共享VM</li>
<li> 将Nginx I/O原语封装(即nginx的API，如时间函数等)后注入Lua VM，允许Lua代码直接访问</li>
<li> 每个外部请求都由一个Lua协程处理，协程之间数据隔离</li>
<li> Lua代码调用I/O操作接口时，若该操作无法立刻完成，则打断相关协程的运行并保护上下文数据，I/O操作完成时还原相关协程上下文数据并继续运行</li>
</ul>
<p style="text-indent: 1em;">简单总结，ngx_lua只是nginx的一个工作模块，它并不是服务；why lua，not php..? nginx在服务过程中，并不会随着请求数的递增，主动派生子进程、子线程。nginx之所以基于固定的几个工作进程即可达到高性能，是因为基于非阻塞的前提，即任何请求不能对nginx造成阻塞。lua原生支持协程机制，使得业务出现阻塞（如等待mysql返回）可以主动放弃执行权，并让nginx继续执行其他请求；直到原来请求执行条件满足（如mysql返回数据），lua可以在原来执行到的地方重新执行。</p>
<p><a id="more"></a><a id="more-317"></a></p>
<h3>ngx_lua vs fastcgi</h3>
<p style="text-indent: 1em;">硬说ngx_lua比后者性能高多少是没有意义的，因为必须结合实际使用场景，否则性能指标显得毫无说服力。所以，我们需要理解什么因素可用让ngx_lua会比cgi高(甚至高得多)</p>
<ul>
<li>fastcgi相对cgi来说，前者是常驻进程，每次请求结束、http连接结束，fastcgi进程并不会马上被web server回收，而是继续等待下个请求、连接的到来，重新工作。由于fastcgi的常驻特性，使得语言编译器可以预加载加载，而减少了像cgi每请求都需要重新加载解析器等工作。所以相对cgi而言，fastcgi性能会得到一定提高（cgi or fastcgi是需要根据业务来衡量）。单fastcgi有个天生的问题，fastcgi进程数决定了系统的并发数，这是对很多互联网业务来说，一个致命的问题。通过增加fastcgi进程数来解决并发的问题是一个得不偿失的方法，根本原因是系统大部分资源是在做进程的来回切换.</li>
</ul>
<ul>
<li>ngx_lua，相对fastcgi来说，性能提升的根本原因，并不只是nginx之于apache(当然nginx也支持fastcgi)，lua之于p*（php,python,perl等p开头语言），更重要的是设计思想的颠覆。cgi、fastcgi的走不出单请求、单连接对应一个系统进程的问题；ngx_lua得益于lua的协程机制，使得每一请求、连接可以对应lua语言级别的协程，对OS来说，对协程的存在是无感知的，实质上是N个请求共用一个系统进程。这使得ngx_lua支撑海量请求成为了可能，而且基于nginx io多路复用的事件模型，并发数的增加，性能稳定。另外ngx_lua的优势可参考<a href="http://slides.dcshi.com/ngx_lua_20130113/#12" target="_blank">slide</a>.</li>
</ul>
<p style="text-indent: 1em;">简单总结，我们的日常场景有哪些阻塞的场景？任何访问远程服务，并且需要回复的，都会是一个阻塞操作，如访问DB，访问cache，访问外部API，所以对于互联网业务老说，阻塞无处不在。所以ngx_lua是很适合对于后台网络密集型的应用场景。值得一提的是，本地的文件IO，还是一个同步的操作，存在阻塞的风险（如磁盘满或者不可写的时候出现）。</p>
<p>&nbsp;<br />
&nbsp;</p>
<p style="text-indent: 1em;">总结，文章从几个角度介绍了ngx_lua，希望能对你理解ngx_lua有帮助。更多可参考<a href="http://openresty.org" target="_blank">http://openresty.org</a>，<a href="https://groups.google.com/forum/?fromgroups#!forum/openresty" target="_blank">google group</a> 或者我曾经在珠三角技术沙龙分享过的一个<a href="http://slides.dcshi.com/ngx_lua_20130113/#1" target="_blank">slide</a>.</p>
