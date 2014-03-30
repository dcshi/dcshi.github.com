---
layout: post
status: publish
published: true
title: ngx_http_qrcode_module nginx二维码模块
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 138
wordpress_url: http://www.dcshi.com/?p=138
date: '2012-12-17 15:35:16 +0000'
date_gmt: '2012-12-17 15:35:16 +0000'
categories:
- nginx
tags: []
---
<p>详细介绍文档请见<a href="https://github.com/dcshi/ngx_http_qrcode_module" target="_blank">github</a></p>
<p>近段时间在阅读nginx的源码，模块是nginx整个生态系统里最重要的部分，所以不得不先了解nginx 模块开发。</p>
<p>二维码作为2012年最热的话题之一，加之前段时间在做的一个应用<a href="http://lomemo.me">lomemo</a>（基于<a href="http://slides.dcshi.com/ngx_lua_20130113/" target="_blank">ngx_lua</a>开发的，推荐web开发的同学了解下）也用到二维码的功能，但当时（现在还在用）是基于ngx_lua实现了<a href="https://github.com/dcshi/lua-resty-QRcode">lua-resty-QRcode</a>（今天她不是主角，改日再写blog介绍好了）。</p>
<p>ngx_http_qrcode_module的实现机制很简单，基于<a href="http://fukuchi.org/works/qrencode/manual/">libqrencode</a>和libgd。前者是封装了二维码的编码实现，后者是生成图片流。此模块的主要工作就是调用前者编码，并通过gd库生成png的图片流，最后返回到client。</p>
<p>这里不讨论nginx模块的开发介绍了，推荐一篇很不错的学习文章:http://www.codinglabs.org/html/intro-of-nginx-module-development.html</p>
<p>正如<a href="https://github.com/dcshi/ngx_http_qrcode_module">github主页</a>文档介绍的一样，此模块支持设置二维码的颜色，大小等，下面通过一个例子来介绍模块的使用。</p>
<p>nginx配置如下:</p>
<pre>worker_processes  1;
daemon off;
error_log  logs/error.log  debug;

events {
	worker_connections  1024;
}

http {
	upstream memcache {
		server localhost:11211;
		keepalive 512;
	}   

	server {
		listen       1314;
		server_name  localhost;

		location /memc {
			internal;

			memc_connect_timeout 100ms;
			memc_send_timeout 100ms;
			memc_read_timeout 100ms;

			set $memc_key $query_string;
			set $memc_exptime 300;

			memc_pass memcache;
		}

		location /qr {
			set $key $uri;
			srcache_fetch GET /memc $key;
			srcache_store PUT /memc $key;

			qrcode_fg_color 000000;
			qrcode_level 0;
			qrcode_hint 2;
			qrcode_size 16;
			qrcode_margin 2;
			qrcode_version 2;
			qrcode_casesensitive off;
			qrcode_txt "http://wwww.dcshi.com";
			qrcode_gen;
		}
	}
}</pre>
<p>其中配置中使用了nginx的另一模块SR Cache (http://wiki.nginx.org/HttpSRCacheModule),把二维码图片流cache在memcache里，避免每次请求重复计算。</p>
<p>上一组性能测试数据，在我的MBA上跑虚拟机(ubuntu ,分配单核），ab测试结果如下：</p>
<pre>ab -c 100 -n 10000 -k "http://localhost:1314/qr"</pre>
<pre>Concurrency Level:      100
Time taken for tests:   1.405 seconds
Complete requests:      10000
Failed requests:        0
Write errors:           0
Keep-Alive requests:    9901
Total transferred:      6789505 bytes
HTML transferred:       5180000 bytes
Requests per second:    7116.42 [#/sec] (mean)
Time per request:       14.052 [ms] (mean)
Time per request:       0.141 [ms] (mean, across all concurrent requests)
Transfer rate:          4718.45 [Kbytes/sec] received</pre>
<p>&nbsp;</p>
